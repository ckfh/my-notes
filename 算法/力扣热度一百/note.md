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
