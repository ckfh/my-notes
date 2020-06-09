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
