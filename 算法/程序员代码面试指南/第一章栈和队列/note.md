# 栈和队列

## 设计一个有 getMin 功能的栈

```Java
public class MyStack1 {
    private Stack<Integer> stackData;
    private Stack<Integer> stackMin;

    public MyStack1() {
        super();
        this.stackData = new Stack<Integer>();
        this.stackMin = new Stack<Integer>();
    }

    public void push(int newNum) {
        if (this.stackMin.isEmpty())
            this.stackMin.push(newNum);
        else if (newNum <= this.getMin())
            this.stackMin.push(newNum);
        this.stackData.push(newNum);
    }

    public int pop() {
        if (this.stackData.isEmpty())
            throw new RuntimeException("Your stack is empty");
        int value = this.stackData.pop();
        if (value == this.getMin())
            this.stackMin.pop();
        return value;
    }

    public int getMin() {
        if (this.stackMin.isEmpty())
            throw new RuntimeException("Your stack is empty.");
        return this.stackMin.peek();
    }
}
```

## 由两个栈实现的队列

```Java
public class TwoStacksQueue {
    private Stack<Integer> stackPush;
    private Stack<Integer> stackPop;

    public TwoStacksQueue() {
        super();
        this.stackPush = new Stack<Integer>();
        this.stackPop = new Stack<Integer>();
    }

    private void pushToPop() {
        if (stackPop.isEmpty())
            while (!stackPush.isEmpty())
                stackPop.push(stackPush.pop());
    }

    public void add(int pushInt) {
        stackPush.push(pushInt);
        this.pushToPop();
    }

    public int poll() {
        if (stackPop.isEmpty() && stackPush.isEmpty())
            throw new RuntimeException("Queue is empty!");
        this.pushToPop();
        return stackPop.pop();
    }

    public int peek() {
        if (stackPop.isEmpty() && stackPush.isEmpty())
            throw new RuntimeException("Queue is empty!");
        this.pushToPop();
        return stackPop.peek();
    }

    public boolean empty() {
        if (stackPop.isEmpty() && stackPush.isEmpty())
            return true;
        return false;
    }
}
```

## 仅用递归逆序一个栈

```Java
public class RecursionReverseStack {
    // 获取栈底元素并移除栈底元素
    public static int getAndRemoveLastElement(Stack<Integer> stack) {
        int result = stack.pop();
        if (stack.isEmpty())
            return result;
        else {
            int last = getAndRemoveLastElement(stack);
            stack.push(result);
            return last;
        }
    }
    // 调用上述方法逆序一个栈
    public static void reverse(Stack<Integer> stack) {
        if (stack.isEmpty())
            return;
        int i = getAndRemoveLastElement(stack); // 获取栈底元素并移除
        reverse(stack);                         // 递归调用
        stack.push(i);                          // 将每次递归调用得到的栈底元素重新压入栈
    }
}
```

## 猫狗队列

```Java
// 以宠物类为属性外加一个时间戳属性定义一个宠物进队类。分别定义猫狗队列，其队列元素类型就是宠物进队类。
// 添加宠物时，如果是狗就打上时间戳进入狗队列，如果是猫就打上时间戳进入猫队列。
// 按照进队先后顺序获取一个宠物时，只要比较两个队列头部元素的时间戳大小，谁在前就从哪个队列出队一个宠物进队对象并获取宠物对象。
```

## 用一个栈实现另一个栈的排序

```Java
public static void sortStackByStack(Stack<Integer> stack) {
    /*
     * 栈一不为空，弹出栈一元素与栈二栈顶元素比较，小于等于则压入栈二，大于则将栈二元素逐一弹出压入栈一，
     * 直到栈二栈顶元素小于等于栈一弹出的元素，最后将栈二元素逐一弹出并压入栈一。
     */
    Stack<Integer> help = new Stack<Integer>();
    while (!stack.isEmpty()) {
        int cur = stack.pop();
        while (!help.isEmpty() && cur > help.peek())
            stack.push(help.pop());
        help.push(cur);
    }
    while (!help.isEmpty()) {
        stack.push(help.pop());
    }
}
```

## 生成窗口最大值数组

```Java
public static int[] getMaxWindow(int[] arr, int w) {
    // 双端队列始终维持一个前大后小的结构，随着窗口右移，队列前面的元素会失效。
    // 维护队列：如果队尾对应的元素小，弹出；如果队尾对应的元素大，就把新的下标加入，继续维护前大后小的结构。
    if (arr == null || w < 1 || arr.length < w)
        return null;
    Deque<Integer> qmax = new LinkedList<Integer>();
    int[] res = new int[arr.length - w + 1];
    int index = 0;
    for (int i = 0; i < arr.length; i++) {
        // 如果队尾对应的元素小，就弹出队尾，直到队尾对应的元素比当前元素大。这一步相当于找当前窗口最大值。
        while (!qmax.isEmpty() && arr[qmax.peekLast()] <= arr[i])
            qmax.pollLast();
        qmax.offerLast(i);
        // 检查队头下标是否过期，过期就把队头弹出，即队头元素不包含在当前窗口。
        if (qmax.peekFirst() == i - w)
            qmax.pollFirst();
        // 如果窗口出现了，就开始记录每个窗口的最大值。
        if (i >= w - 1)
            res[index++] = arr[qmax.peekFirst()];
    }
    return res;
}
```
