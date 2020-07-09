# 泛型

泛型是一种“代码模板”，可以用一套代码**套用各种类型**。

## 什么是泛型

在Java标准库中的`ArrayList<T>`实现了`List<T>`接口，它可以向上转型为`List<T>`，即类型`ArrayList<T>`可以向上转型为`List<T>`。但是，不能把`ArrayList<Integer>`向上转型为`ArrayList<Number>`或`List<Number>`。

```Java
// 创建ArrayList<Integer>类型：
ArrayList<Integer> integerList = new ArrayList<Integer>();
// 添加一个Integer：
integerList.add(new Integer(123));
// “向上转型”为ArrayList<Number>：
ArrayList<Number> numberList = integerList;
// 添加一个Float，因为Float也是Number：
numberList.add(new Float(12.34));
// 从ArrayList<Integer>获取索引为1的元素（即添加的Float）：
Integer n = integerList.get(1); // ClassCastException!
```

**`ArrayList<Integer>`和`ArrayList<Number>`两者完全没有继承关系**。

## 使用泛型

使用ArrayList时，如果不定义泛型类型时，泛型类型实际上就是Object，此时，只能把`<T>`当作Object使用，没有发挥泛型的优势。

除了`ArrayList<T>`使用了泛型，还可以在接口中使用泛型。例如，Arrays.sort(Object[])可以对任意数组进行排序，但待排序的元素必须实现`Comparable<T>`这个泛型接口。

## 编写泛型

编写泛型类比普通类要复杂。通常来说，**泛型类一般用在集合类中**，例如`ArrayList<T>`，**我们很少需要编写泛型类**。

可以按照以下步骤来编写一个泛型类。

```Java
// 首先，按照某种类型，例如：String，来编写类：
public class Pair {
    private String first;
    private String last;
    public Pair(String first, String last) {
        this.first = first;
        this.last = last;
    }
    public String getFirst() {
        return first;
    }
    public String getLast() {
        return last;
    }
}
// 然后，标记所有的特定类型，这里是String：
// 最后，把特定类型String替换为T，并申明<T>（在IDEA中，选中String，Ctrl+R，替换内容为T，全部替换）：
public class Pair<T> {
    private T first;
    private T last;
    public Pair(T first, T last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() {
        return first;
    }
    public T getLast() {
        return last;
    }
}
```

编写泛型类时，要特别注意，泛型类型`<T>`不能用于静态方法。我们无法在静态方法create()的**方法参数**和**返回类型**上使用泛型类型T。

```Java
public class Pair<T> {
    ...
    // 对静态方法使用<T>：
    // 编译错误：'xxx.Pair.this' cannot be referenced from a static context：
    public static Pair<T> create(T first, T last) {
        return new Pair<T>(first, last);
    }
}
public class Pair<T> {
    ...
    // 在static修饰符后面加一个<T>，编译就能通过：
    // 但实际上，这个<T>和Pair<T>类型的<T>已经没有任何关系了：
    public static <T> Pair<T> create(T first, T last) {
        return new Pair<T>(first, last);
    }
}
public class Pair<T> {
    ...
    // 对于静态方法，我们可以单独改写为“泛型”方法，只需要使用另一个类型即可：
    // 静态泛型方法应该使用其它类型区分:
    public static <K> Pair<K> create(K first, K last) {
        return new Pair<K>(first, last);
    }
}
```

泛型还可以定义多种类型。Java标准库的`Map<K, V>`就是使用两种泛型类型的例子。它对Key使用一种类型，对Value使用另一种类型。

## 擦拭法

Java语言的泛型实现方式是**擦拭法**。所谓擦拭法是指，虚拟机对泛型其实一无所知，所有的工作都是编译器做的。

```Java
// 这是编译器看到的代码：
public class Pair<T> {
    private T first;
    private T last;
    public Pair(T first, T last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() {
        return first;
    }
    public T getLast() {
        return last;
    }
}
// 这是虚拟机执行的代码：
public class Pair {
    private Object first;
    private Object last;
    public Pair(Object first, Object last) {
        this.first = first;
        this.last = last;
    }
    public Object getFirst() {
        return first;
    }
    public Object getLast() {
        return last;
    }
}
```

因此，Java使用擦拭法实现泛型，导致了：

- **编译器把类型`<T>`视为Object**；
- **编译器根据`<T>`实现安全的强制转型**。

```Java
// 使用泛型的时候，我们编写的代码也是编译器看到的代码：
Pair<String> p = new Pair<>("Hello", "world");
String first = p.getFirst();
String last = p.getLast();
// 而虚拟机执行的代码并没有泛型：
Pair p = new Pair("Hello", "world");
String first = (String) p.getFirst();
String last = (String) p.getLast();
```

所以，Java的泛型是由编译器在编译时实行的，编译器内部永远把所有类型T视为Object处理，但是，在需要转型的时候，编译器会根据T的类型自动为我们实行安全的强制转型。

了解了Java泛型的实现方式——擦拭法，我们就知道了Java泛型的局限：

```Java
// 局限一：<T>不能是基本类型，例如int，因为实际类型是Object，Object类型无法持有基本类型：
Pair<int> p = new Pair<>(1, 2); // compile error!

// 局限二：无法取得带泛型的Class，所有泛型实例，无论T的类型是什么，getClass()返回同一个Class实例，因为编译后它们全部都是Pair<Object>：
public class Main {
    public static void main(String[] args) {
        Pair<String> p1 = new Pair<>("hello", "world");
        Pair<Integer> p2 = new Pair<>(123, 456);
        Class c1 = p1.getClass();
        Class c2 = p2.getClass();
        System.out.println(c1 == c2);         // true
        System.out.println(c1 == Pair.class); // true
    }
}

// 局限三：无法判断带泛型的类型，原因和前面一样，并不存在Pair<String>.class，而是只有唯一的Pair.class：
Pair<Integer> p = new Pair<>(123, 456);
// Compile error:
if (p instanceof Pair<String>) {
}

// 局限四：不能实例化T类型，创建new Pair<String>()和创建new Pair<Integer>()就全部成了Object，显然编译器要阻止这种类型不对的代码。：
public class Pair<T> {
    private T first;
    private T last;
    public Pair() {
        // Compile error:
        first = new T(); // 擦拭后变成了new Object()
        last = new T();  // 擦拭后变成了new Object()
    }
}
// 要实例化T类型，我们必须借助额外的Class<T>参数：
public Pair(Class<T> clazz) {
    this.first = clazz.newInstance();
    this.last = clazz.newInstance();
}
// 上述代码借助Class<T>参数并通过反射来实例化T类型，使用的时候，也必须传入Class<T>。例如：
// 因为传入了Class<String>的实例，所以我们借助String.class就可以实例化String类型：
Pair<String> pair = new Pair<>(String.class);
```

有些时候，一个看似正确定义的方法会无法通过编译，编译器会**阻止一个实际上会变成覆写的泛型方法定义**。

```Java
public class Pair<T> {
    public boolean equals(T t) {
        return this == t;
    }
}
// 这是因为，定义的equals(T t)方法实际上会被擦拭成equals(Object t)，而这个方法是继承自Object的，编译器会阻止一个实际上会变成覆写的泛型方法定义。
// 换个方法名，避开与Object.equals(Object)的冲突就可以成功编译：
public class Pair<T> {
    public boolean same(T t) {
        return this == t;
    }
}
```

一个类可以继承自一个泛型类。

```Java
// 例如：父类的类型是Pair<Integer>，子类的类型是IntPair，可以这么继承：
public class IntPair extends Pair<Integer> {
}
// 使用的时候，因为子类IntPair并没有泛型类型，所以，正常使用即可：
IntPair ip = new IntPair(1, 2);
```

前面讲了，我们无法获取`Pair<T>`的T类型，即给定一个变量`Pair<Integer> p`，无法从p中获取到Integer类型。但是，**在父类是泛型类型的情况下，编译器就必须把类型T（对IntPair来说，也就是Integer类型）保存到子类的class文件中，不然编译器就不知道IntPair只能存取Integer这种类型**。

在继承了泛型类型的情况下，子类可以获取父类的泛型类型。

```Java
public class Main {
    public static void main(String[] args) {
        Class<IntPair> clazz = IntPair.class;
        Type t = clazz.getGenericSuperclass();
        if (t instanceof ParameterizedType) {
            ParameterizedType pt = (ParameterizedType) t;
            Type[] types = pt.getActualTypeArguments(); // 可能有多个泛型类型
            Type firstType = types[0];                  // 取第一个泛型类型
            Class<?> typeClass = (Class<?>) firstType;
            System.out.println(typeClass);              // Integer
        }
    }
}
```

因为Java引入了泛型，所以，只用Class来标识类型已经不够了。实际上，Java的类型系统结构如下：

<img src="./image/类型系统结构.jpg"/>

## extends通配符
