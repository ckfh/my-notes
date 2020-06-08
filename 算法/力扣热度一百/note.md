# 热度前一百

## 两数相加

```Java
// 节点值加上进位值除以十赋给进位值，取余十赋给节点值
// 注意，一定要先让“指针”指向一个节点，再让“指针”往后走
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode ans = new ListNode(0);
    ListNode p = ans;
    int carry = 0;
    while (l1 != null && l2 != null) {
        int sum = (l1.val + l2.val + carry);
        carry = sum / 10;
        p.next = new ListNode(sum % 10);
        l1 = l1.next;
        l2 = l2.next;
        p = p.next;
    }
    while (l1 != null) {
        int sum = (l1.val + carry);
        carry = sum / 10;
        p.next = new ListNode(sum % 10);
        l1 = l1.next;
        p = p.next;
    }
    while (l2 != null) {
        int sum = (l2.val + carry);
        carry = sum / 10;
        p.next = new ListNode(sum % 10);
        l2 = l2.next;
        p = p.next;
    }
    if (carry == 1) {
        p.next = new ListNode(carry);
    }
    return ans.next;
}
```

## 无重复字符的最长子串

```Java
// 维护一个滑动窗口，以起始字符为左边界，不断向右扩大窗口范围。
// 扩大的依据是当前字符不在该无重复字符子串中；缩小的依据是当前字符已存在，需要不断改变左边界的值直到移除之前的重复字符。
public int lengthOfLongestSubstring(String s) {
    Set<Character> occ = new HashSet<>();
    int n = s.length();
    int rk = -1, ans = 0;
    for (int i = 0; i < n; i++) {
        if (i != 0)
            occ.remove(s.charAt(i - 1));
        while (rk + 1 < n && !occ.contains(s.charAt(rk + 1))) {
            occ.add(s.charAt(rk + 1));
            rk++;
        }
        ans = Math.max(ans, rk - i + 1);
    }
    return ans;
}
```
