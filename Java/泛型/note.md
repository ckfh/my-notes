# 泛型

泛型是一种“代码模板”，可以用一套代码套用各种类型。

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

Java语言的泛型实现方式是**擦拭法**（Type Erasure）。所谓擦拭法是指，虚拟机对泛型其实一无所知，所有的工作都是编译器做的。

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

- 编译器把类型`<T>`视为Object；
- 编译器根据`<T>`实现安全的强制转型。
