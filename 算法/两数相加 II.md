# 两数相加 II

## 题目描述

给你两个 非空 链表来代表两个非负整数。数字最高位位于链表开始位置。它们的每个节点只存储一位数字。将这两数相加会返回一个新的链表。

你可以假设除了数字 0 之外，这两个数字都不会以零开头。

**进阶：**

如果输入链表不能修改该如何处理？换句话说，你不能对列表中的节点进行翻转。

**示例：**

输入：(7 -> 2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 8 -> 0 -> 7

## 方法一：将链表倒置，然后倒置组装

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode ll1 = reverse(l1);
        ListNode ll2 = reverse(l2);
        ListNode root = null, oldRoot = null;
        int c = 0;
        while (ll1 != null && ll2 != null) {
            oldRoot = root;
            root = new ListNode((ll1.val + ll2.val + c) % 10);
            c = (ll1.val + ll2.val + c) / 10;
            root.next = oldRoot;
            ll1 = ll1.next; ll2 = ll2.next;
        }
        while (ll1 != null) {
            oldRoot = root;
            root = new ListNode((ll1.val + c) % 10);
            c = (ll1.val + c) / 10;
            root.next = oldRoot;
            ll1 = ll1.next;
        }
        while (ll2 != null) {
            oldRoot = root;
            root = new ListNode((ll2.val + c) % 10);
            c = (ll2.val + c) / 10;
            root.next = oldRoot;
            ll2 = ll2.next;
        }
        if (c != 0) {
            oldRoot = root;
            root = new ListNode(c);
            root.next = oldRoot;
        }
        return root;
    }

    private ListNode reverse(ListNode l) {
        ListNode root = null, oldRoot = null;
        while (l != null) {
            oldRoot = root;
            root = new ListNode(l.val);
            root.next = oldRoot;
            l = l.next;
        }
        return root;
    }
}
```

## 方法二：栈

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        Stack<Integer> s1 = new Stack<>();
        Stack<Integer> s2 = new Stack<>();
        addAll(s1, l1);
        addAll(s2, l2);
        ListNode root = null, oldRoot = null;
        int carry = 0;
        while (!s1.isEmpty() || !s2.isEmpty() || carry > 0) {
            int a = s1.isEmpty()? 0: s1.pop();
            int b = s2.isEmpty()? 0: s2.pop();
            int cur = a + b + carry;
            carry = cur / 10;
            cur %= 10;
            oldRoot = root;
            root = new ListNode(cur);
            root.next = oldRoot;
        }
        return root;
    }

    private void addAll(Stack<Integer> s, ListNode l) {
        while (l != null) {
            s.add(l.val);
            l = l.next;
        }
    }
}
```
