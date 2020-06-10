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
