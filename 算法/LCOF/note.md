# LCOF

## 03 数组中重复的数字

```Java
public int findRepeatNumber(int[] nums) {
    int[] ansArray = new int[nums.length];
    for (int num : nums) {
        ansArray[num] += 1;
        if (ansArray[num] >= 2)
            return num;
    }
    throw new RuntimeException("No duplicate numbers");
}
```

## 04 二维数组中的查找

```Java
public boolean findNumberIn2DArray(int[][] matrix, int target) {
    for (int[] array : matrix) {
        if (Arrays.binarySearch(array, target) >= 0)
            return true;
    }
    return false;
}
```

## 05 替换空格

```Java
public String replaceSpace(String s) {
    return s.replace(" ", "%20");
}
```

## 06 从尾到头打印链表

```Java
public int[] reversePrint(ListNode head) {
    int len = 0;
    for (ListNode p = head; p != null; p = p.next)
        len++;
    int[] ans = new int[len];
    for (ListNode p = head; p != null; p = p.next)
        ans[--len] = p.val;
    return ans;
}
```

## 07 重建二叉树

```Java
public TreeNode buildTree(int[] preorder, int[] inorder) {
    if (preorder == null || preorder.length == 0) {
        return null;
    }
    Map<Integer, Integer> indexMap = new HashMap<Integer, Integer>();
    int length = preorder.length;
    for (int i = 0; i < length; i++) {
        indexMap.put(inorder[i], i);
    }
    TreeNode root = buildTree(preorder, 0, length - 1, inorder, 0, length - 1, indexMap);
    return root;
}
public TreeNode buildTree(int[] preorder, int preorderStart, int preorderEnd, int[] inorder, int inorderStart, int inorderEnd, Map<Integer, Integer> indexMap) {
    if (preorderStart > preorderEnd) {
        return null;
    }
    int rootVal = preorder[preorderStart];
    TreeNode root = new TreeNode(rootVal);
    if (preorderStart == preorderEnd) {
        return root;
    } else {
        int rootIndex = indexMap.get(rootVal);
        int leftNodes = rootIndex - inorderStart, rightNodes = inorderEnd - rootIndex;
        TreeNode leftSubtree = buildTree(preorder, preorderStart + 1, preorderStart + leftNodes, inorder, inorderStart, rootIndex - 1, indexMap);
        TreeNode rightSubtree = buildTree(preorder, preorderEnd - rightNodes + 1, preorderEnd, inorder, rootIndex + 1, inorderEnd, indexMap);
        root.left = leftSubtree;
        root.right = rightSubtree;
        return root;
    }
}
```

## 09 用两个栈实现队列

```Java
public class CQueue {
    private final Deque<Integer> inStack;
    private final Deque<Integer> outStack;

    public CQueue() {
        this.inStack = new ArrayDeque<>();
        this.outStack = new ArrayDeque<>();
    }

    private void inToOut() {
        if (this.outStack.isEmpty())
            while (!this.inStack.isEmpty())
                this.outStack.push(this.inStack.pop());
    }

    public void appendTail(int value) {
        this.inStack.push(value);
        this.inToOut();
    }

    public int deleteHead() {
        if (this.outStack.isEmpty() && this.inStack.isEmpty())
            return -1;
        this.inToOut();
        return this.outStack.pop();
    }
}
```

## 10.1 斐波那契数列

```Java
public int fib(int n) {
    if (n == 0)
        return 0;
    int first = 1;
    int second = 1;
    for (int i = 2; i < n; i++) {
        int third = first + second;
        first = second;
        second = third % 1000000007;
    }
    return second;
}
```

## 10.2 青蛙跳台阶问题

```Java
public int numWays(int n) {
    if (n == 0)
        return 1;
    int first = 1;
    int second = 1;
    for (int i = 2; i <= n; i++) {
        int third = first + second;
        first = second;
        second = third % 1000000007;
    }
    return second;
}
```
