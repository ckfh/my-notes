# 记录我自己踩过又老是忘记的坑

## Maven 项目配置

- 远程仓库修改、本地仓库修改、JDK 版本修改。
- Eclipse IDE 配置：增加本地 Maven 安装目录、修改全局配置文件地址。
- 下载 archetype-catalog 文件，增加到 IDE 的 catelog 文件目录当中。
- 新建 Maven 项目时，选择 org.apache.maven.archetypes 所属的 maven-archetype-quickstart 原型进行创建。

## 值传递和引用传递

- [掘金文章](https://juejin.im/post/5bce68226fb9a05ce46a0476)
- 对基本数据类型的数据进行操作，原始内容和副本都是存储实际值，但是两者是在不同的栈区，因此对形参的修改不会影响到原始内容。

    ```Java
    public static void main(String[] args) {
        int x = 20;
        System.out.printf("main() x = %d\n", x); // main() x = 20
        foo(x);
        System.out.printf("main() x = %d\n", x); // main() x = 20
    }
    static void foo(int x) {
        System.out.printf("foo() x = %d\n", x); // foo() x = 20
        x = 10;
        System.out.printf("foo() x = %d\n", x); // foo() x = 10
    }
    ```

- Java中任何引用类型变量的赋值内容从来都是一块堆内存的地址，所以不要妄想做出先用该变量指向一个堆内存地址，然后再指向另外一个堆内存地址，以期望当对新地址的堆内存内容进行修改时能够改变旧地址指向的堆内存内容这样的做法。

    ```Java
    public static void main(String[] args) {
        Person per = new Person();
        per.setName("cat");
        System.out.println(per); // Person [name=cat]
        // 传递地址值
        foo(per);
        System.out.println(per); // Person [name=dog]
    }
    static void foo(Person per) {
        System.out.println(per); // Person [name=cat]
        // 没有发生改变地址值的操作，因此修改会改变实参指向的对象内容
        per.setName("dog");
        System.out.println(per); // Person [name=dog]
    }
    ```

    ```Java
    public static void main(String[] args) {
        Person per = new Person();
        per.setName("cat");
        System.out.println(per); // Person [name=cat]
        // 传递地址值
        foo(per);
        System.out.println(per); // Person [name=cat]
    }
    static void foo(Person per) {
        System.out.println(per); // Person [name=cat]
        // 注意此时改变了实参传给形参的地址值，因此修改并不会影响原来实参指向的对象内容
        per = new Person();
        per.setName("dog");
        System.out.println(per); // Person [name=dog]
    }
    ```
