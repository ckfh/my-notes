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
