# 码上精进算法21天打卡活动·Day 1		
			

## 关于数组基础及二分查找及其知识点 

### 二分搜索算法核心代码模板

上面的迭代解法操作指针虽然有些繁琐，但是思路还是比较清晰的。如果现在让你用递归来反转单链表，你有啥想法没？

对于初学者来说可能很难想到，这很正常。如果你学习了后文的二叉树系列算法思维，回头再来看这道题，才有可能自己想出这个算法。

因为二叉树结构本身就是单链表的延伸，相当于是二叉链表嘛，所以二叉树上的递归思维，套用到单链表上是一样的。

递归反转单链表的关键在于，这个问题本身是存在子问题结构的。

比方说，现在给你输入一个以 1 为头结点单链表 1->2->3->4，那么如果我忽略这个头结点 1，只拿出 2->3->4 这个子链表，它也是个单链表对吧？

那么你这个 reverseList 函数，只要输入一个单链表，就能给我反转对吧？那么你能不能用这个函数先来反转 2->3->4 这个子链表呢，然后再想办法把 1 接到反转后的 4->3->2 的最后面，是不是就完成了整个链表的反转？

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        ListNode* prev = nullptr;   // 前驱节点
        ListNode* curr = head;      // 当前节点
        
        while (curr != nullptr) {
            ListNode* nextTemp = curr->next; // 1. 暂存下一个节点
            curr->next = prev;               // 2. 反转当前节点的指针
            prev = curr;                     // 3. prev 前进
            curr = nextTemp;                 // 4. curr 前进
        }
        
        return prev; // 循环结束时 prev 指向新的头节点
    }
};
```