

# 腾讯高频351总结



## 链表

**链表相关题目的常用操作就是设置虚拟头结点，递归解决，快慢指针（判环，找中点）**

### 876. 链表的中间结点

思路：快慢指针



### 142、环形链表2

思路：需要一点数学计算，最终结论是快慢指针相遇时，再启动一个新指针从头和slow一起走，当它们相遇时，该节点即为环入口

```cpp
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        ListNode *slow = head, *fast = head;
        while (fast != nullptr && fast->next != nullptr) {
            fast = fast->next->next;
            slow = slow->next;
            if (fast == slow) {
                ListNode *p = head;
                while (p != slow) {
                    p = p->next;
                    slow = slow->next;
                }
                return p;
            }
        }
        return nullptr;
    }
};
```



### 160、相交链表/LCR171

为什么下面这样不行？

```cpp
while (pa != pb) {
    pa = pa->next;
    if (pa == nullptr) pa = headB;
    pb = pb->next;
    if (pb == nullptr) pb = headA;
}
```

因为如果链表不相交时，这样会陷入死循环，因为无法让pa=pb=nullptr的情况出现，这样情况即为不相交

```cpp
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        ListNode *pa = headA, *pb = headB;
        while (pa != pb) {
            if (pa == nullptr) pa = headB;
            else pa = pa->next;
            if (pb == nullptr) pb = headA;
            else pb = pb->next;
        }
        return pa;
    }
};
```



### 19、删除链表的倒数第N个节点

### LCR140、返回链表倒数第N个节点

这题是19题的简易版本，删除链表中的一个节点需要遍历到它的前一个节点，所以19题需要一个虚拟头结点来简化代码和思路。



### 83、删除链表中的重复元素（保留一个）

### 82、删除链表中的重复元素2（只要是重复的元素，全部删除）

思路：新建一个虚拟头结点

遍历过程中记录pre节点，cur节点，cur->next == nullptr表示当前只剩一个元素，所以不需要再判断了；

对于每一个cur，新建一个p指针，遍历后面的元素，同时记录have_same表示是否有重复，有重复时需要将

`pre->next = p;cur = p;`，无重复时，继续向后遍历即可

```cpp
class Solution {
public:
    ListNode* deleteDuplicates(ListNode* head) {
        if (head == nullptr || head->next == nullptr) return head;
        ListNode *preHead = new ListNode(-1, head);

        ListNode *pre = preHead, *cur = head;
        while (cur != nullptr && cur->next != nullptr) {
            ListNode *p = cur->next;
            bool have_same = false;
            while (p != nullptr && p->val == cur->val) {
                have_same = true;
                p = p->next;
            }
            if (have_same) {
                pre->next = p;
                cur = p;
            } else {
                cur = cur->next;
                pre = pre->next;
            }
            
        }
        return preHead->next;
    }
};
```





### 24、两两交换链表中的节点

递归：

主函数中处理只有0/1个节点的情况，然后递归交换节点；

在递归交换函数中，递归出口为cur为空，表示后面没节点了或者只有一个节点不够交换；交换两个节点；递归交换时需要用到`pre->next->next`，所以判断一下pre->next是否为空，如果为空，则表示后面没节点了，直接返回cur即可，不为空时递归交换一下再返回cur；

```cpp
class Solution {
public:
    ListNode* swapPairs(ListNode* head) {
        if (head == nullptr || head->next == nullptr) return head;
        ListNode* pre = head, *cur = head->next;
        head = swapTwoNode(pre, cur);
        return head;
    }

    ListNode* swapTwoNode(ListNode*pre, ListNode* cur) {
        if (cur == nullptr) return pre;
        pre->next = cur->next;
        cur->next = pre;
        if (pre->next == nullptr) return cur;
        pre->next = swapTwoNode(pre->next, pre->next->next);
        return cur;
    }
};
```

迭代：

```cpp
class Solution {
public:
    ListNode* swapPairs(ListNode* head) {
        if (head == nullptr || head->next == nullptr) return head;
        ListNode *preHead = new ListNode(-1);
        ListNode *pre = preHead, *cur = head;
        while (cur != nullptr && cur->next != nullptr) {
            pre->next = cur->next;
            cur->next = cur->next->next;
            pre->next->next = cur;
            pre = cur;
            cur = cur->next;
        }
        return preHead->next;   
    }
};
```

### 25、K个一组翻转链表

思路：使用递归写法，先遍历判断是否有k个元素，不足k个则为递归出口，返回head即可，如果够k个，则对这k个进行链表翻转，然后递归的翻转后面的链表，最后这两步要记住`preHead->next->next = reverseKGroup(cur, k);	return pre;`，

```c++
class Solution {
public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        int count = 0;
        ListNode* p = head;
        while (p != nullptr) {
            count++;
            if (count == k) break;
            p = p->next;
        }
        if (count != k) return head;
        ListNode* preHead = new ListNode(-1);
        preHead->next = head;
        ListNode* pre = preHead, *cur = head;
        while (count--) {
            ListNode *ne = cur->next;
            cur->next = pre;
            pre = cur;
            cur = ne;
        }
        preHead->next->next = reverseKGroup(cur, k);
        return pre;
    }
};
```

### 138、随机链表的复制

思路：用哈希表来存储旧节点和新拷贝的节点，用来表示该节点是否深拷贝过，如果深拷贝过，直接用哈希表返回即可，没有深拷贝过则拷贝这个节点，并设置它的两个指针。

```cpp
class Solution {
public:
    unordered_map<Node*, Node*> cache_node;

    Node* copyRandomList(Node* head) {
        if (head == nullptr) return head;
        
        if (cache_node.count(head) == 0) {
            Node* head_copy = new Node(head->val);
            cache_node[head] = head_copy;
            if (head->next) head_copy->next = copyRandomList(head->next);
            if (head->random) head_copy->random = copyRandomList(head->random);
        }

        return cache_node[head];
    }
};
```



### 147、链表插入排序

思路：

设置虚拟头结点，保存premax:上次最大的节点，cur:当前需要插入到前面的节点

使用pre=preHead往后遍历，直到后一个元素为cur或者后一个元素的值小于等于cur的值（在前面找到需要插入的位置）；

如果下一个节点为cur，则说明cur不需要移动，cur为当前最大元素；

如果下一个节点不是cur，则需要移动节点；

```c++
class Solution {
public:
    ListNode* insertionSortList(ListNode* head) {
        ListNode *preHead = new ListNode(-1);
        preHead->next = head;
        ListNode *premax = head;
        ListNode *cur = head->next;

        while (cur != nullptr) {
            // 找到第一个大于cur的位置
            ListNode *pre = preHead;
            while (pre->next != cur && pre->next->val <= cur->val) {
                pre = pre->next;
            }
            // cur为最大，不需要移动
            if (pre->next == cur) {
                premax = cur;
                cur = cur->next;
                continue;
            }
            // 移动cur，此时premax不需要改变
            ListNode *ne = cur->next;
            cur->next = pre->next;
            pre->next = cur;
            premax->next = ne;
            cur = ne;
        }
        return preHead->next;
    }
};
```

### 148、链表归并排序

```cpp
class Solution {
public:
    ListNode* sortList(ListNode* head) {
        // 递归返回入口
        if (head == nullptr || head->next == nullptr) return head;

        // 归
        ListNode *slow = head, *fast = head->next;
        while (fast != nullptr && fast->next != nullptr) {
            fast = fast->next->next;
            slow = slow->next;
        }
        ListNode *right_head = slow->next;
        slow->next = nullptr;
        ListNode *l = sortList(head);
        ListNode *r = sortList(right_head);

        // 并
        ListNode *preHead = new ListNode(-1);
        ListNode *cur = preHead;
        while (l != nullptr && r != nullptr) {
            if (l->val < r->val) {
                cur->next = l;
                l = l->next;
            } else {
                cur->next = r;
                r = r->next;
            }
            cur = cur->next;
        }
        if (l != nullptr) cur->next = l;
        if (r != nullptr) cur->next = r;
        return preHead->next;
    }
};
```



### 206、反转链表1

```cpp
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        if (head == nullptr) return head;
        ListNode *preHead = new ListNode(-1);
        preHead->next = head;
        ListNode *cur = head, *pre = preHead;
        while (cur != nullptr) {
            ListNode *tmp = cur;
            cur = cur->next;
            tmp->next = pre;
            pre = tmp;
        }
        preHead->next->next = nullptr;
        return pre;
    }
};
```

### 92、反转链表2

```cpp
class Solution {
public:
    ListNode* reverseBetween(ListNode* head, int left, int right) {
        ListNode *preHead = new ListNode(-1);
        preHead->next = head;

        ListNode *pre = preHead;
        ListNode *cur = head;
        int cnt = 1;
        while (cnt != left && cur != nullptr) {
            pre = pre->next;
            cur = cur->next;
            cnt++;
        }
        ListNode *preleft = pre;
        while (cnt <= right && cur != nullptr) {
            ListNode *ne = cur->next;
            cur->next = pre;
            pre = cur;
            cur = ne;
            cnt++;
        }
        preleft->next->next = cur;
        preleft->next = pre;
        return preHead->next;
    }
};
```

### 61、旋转链表

思路：

首先将k对len取模，当k == 0时直接返回，链表只有0/1个节点也直接返回；

找到链表的倒数第k + 1个节点，将cur指向它，将rear指向链表尾元素

然后开始调整位置，记得设置`cur->next = nullptr;`

```cpp
class Solution {
public:
    ListNode* rotateRight(ListNode* head, int k) {
        if (head == nullptr || head->next == nullptr) return head;
        // k > len时取模
        int len = 0;
        ListNode *p = head;
        while (p != nullptr) {
            p = p->next;
            len++;
        }
        k %= len;
        if (k == 0) return head;

        // 找到倒数第k+1个节点
        ListNode *rear = head, *cur = head;
        while (k--) rear = rear->next;
        while (rear->next != nullptr) {
            rear = rear->next;
            cur = cur->next;
        }

        // 调整位置
        ListNode *tmp = head;
        head = cur->next;
        cur->next = nullptr;
        rear->next = tmp;

        return head;
    }
};
```



### 链表题目总结

复习时间：

- [x] 7.11
- [ ] 7.12
- [ ] 7.14
- [ ] 7.18
- [ ] 7.26
- [ ] 8.10

**反转链表**

**反转链表2**：在给定left~right内反转链表

**两两交换链表的节点**：递归写法：返回入口(head或者head->next为空返回head)，获取三个节点（head,head->next,head->next->next），node1的next为反转后的node3，node2的next为node1，返回node2即可

```cpp
ListNode *swapPairs(ListNode *head) {
        if (head == nullptr || head->next == nullptr)
            return head;

        auto node1 = head;
        auto node2 = head->next;
        auto node3 = node2->next;

        node1->next = swapPairs(node3); // 1 指向递归返回的链表头
        node2->next = node1; // 2 指向 1

        return node2; // 返回交换后的链表头节点
    }
```

迭代法：

```cpp
ListNode *swapPairs(ListNode *head) {
        auto dummy = new ListNode(0, head); // 用哨兵节点简化代码逻辑
        auto node0 = dummy;
        auto node1 = head;
        while (node1 && node1->next) { // 至少有两个节点
            auto node2 = node1->next;
            auto node3 = node2->next;

            node0->next = node2; // 0 -> 2
            node2->next = node1; // 2 -> 1
            node1->next = node3; // 1 -> 3

            node0 = node1; // 下一轮交换，0 是 1
            node1 = node3; // 下一轮交换，1 是 3
        }
        return dummy->next; // 返回新链表的头节点
    }
```

**K个一组反转链表**：判断是否够k个，够的话，反转k个，然后递归反转后面的（处理好反转完两个k段之间的关系）

```cpp
preHead->next->next = reverseKGroup(cur, k);
return pre;
```

**旋转链表**：先将k对链表长度取模，然后找到倒数第k+1个节点和尾节点，然后调整三个节点（head，rear，cur）的关系即可

**奇偶链表**：先将偶数节点分离出来，然后再接到奇数节点的尾部

```cpp
// 分离奇偶节点
ListNode *evenPreHead = new ListNode(-1);
ListNode *cur = head;
ListNode *evenCur = evenPreHead;
while (cur != nullptr && cur->next != nullptr) {
    evenCur->next = cur->next;
    evenCur = evenCur->next;
    cur->next = cur->next->next;
    cur = cur->next;
}
evenCur->next = nullptr;
```

**重排链表**：先快慢指针找中点分离后半段，再反转后半段，最后合并两段

```cpp
// 交叉合并两段链表
ListNode *p = head, *q = rightHead;
while (p != nullptr && q != nullptr) {
    ListNode *nep = p->next;
    p->next = q;
    p = nep;
    ListNode *neq = q->next;
    q->next = nep;
    q = neq;
}	
```



**链表的中间节点**

**回文链表**：快慢指针找中点进行分割，反转后半段链表，然后对比数值



**两数相加1**：记住简化版的代码，迭代方式创建新节点更方便，递归方式可使用原节点，最好用迭代方式

```cpp
ListNode *preHead = new ListNode(0);
auto *cur = preHead;
int carry = 0;
while (carry || l1 || l2) {
    int cursum = (l1 ? l1->val : 0) + (l2 ? l2->val : 0) + carry;
    cur->next = new ListNode(cursum % 10);
    cur = cur->next;
    carry = cursum / 10;
    if (l1) l1 = l1->next;
    if (l2) l2 = l2->next;
}
```

**两数相加2**：两数相加1结合反转链表

**合并两个有序链表**

**合并k个有序链表**：使用优先队列



**环形链表**：快慢指针判断是否有环

**环形链表2**：快慢指针后，再添1个慢指针与原来慢指针相遇即为入口点

**相交链表**：两个指针从两个起点往后遍历，遍历到底再从对方的起点开始遍历，相遇时即为交点，如果两者为空，说明没相遇，所以我们需要代码中判断两者都为空的情况出现

```cpp
while (pa != pb) {
    if (pa == nullptr) pa = headB;
    else pa = pa->next;
    if (pb == nullptr) pb = headA;
    else pb = pb->next;
}
```



**链表中倒数第k个节点**：两个指针一起遍历，一个先走k步，然后再一起走（举一反三：找倒数第k-1，可以使先走指针的下一节点不为空）

**删除链表中的倒数第k个节点**：找倒数第k+1个



**删除链表中的重复元素**

**删除链表中的重复元素2**：使用bool值have_same记录是否有重复，cur指针遍历当前需要判断的值，pre指针用于当重复元素出现需要删除它们时使用



**排序链表（归并排序**）：递归入口、归（快慢指针找中点分割链表）、并（合并两个升序链表）

**链表的插入排序**：n轮遍历，保存上一次的最大值指针premax和当前遍历的指针cur，如果当前值为最大值，调整premax=cur，如果不是，则调整premax->next



**复制带随机指针的链表**：递归写，用哈希表维护已深拷贝的节点





## 树

**二叉树的题目需要掌握常用的几种遍历手段和通过BFS和DFS达成某些目的**



### 基础遍历方式

94、二叉树的中序遍历

144、二叉树的前序遍历

145、二叉树的后序遍历

102、二叉树的层序遍历

剑指54、二叉搜索树的第k大节点（中序遍历到数组中即可）



### BFS型

107、二叉树的层序遍历2

103、二叉树的锯齿形遍历

429、N叉树的层序遍历



515、在每个树行中找最大值

637、二叉树的层平均值

199、二叉树的右视图



116、填充每个节点的下一个右侧节点指针

117、填充每个节点的下一个右侧节点指针



111、二叉树的最小深度（DFS和BFS均可）

104、二叉树的最大深度（DFS和BFS均可）



### DFS型（递归型）

二叉树的最大深度

N叉树的最大深度

二叉树的最小深度

二叉树的最大宽度

```cpp
class Solution {
public:
    int ans = 0;
    unordered_map<int, int> map;
    int widthOfBinaryTree(TreeNode* root) {
        dfs(root, 1, 0);
        return ans;
    }

    void dfs(TreeNode *root, int u, int d) {
        if (root == nullptr) return;
        if (!map.count(d)) map[d] = u;
        ans = max(ans, u - map[d] + 1);
        u = u - map[d] + 1;
        dfs(root->left, u << 1, d + 1);
        dfs(root->right, u << 1 | 1, d + 1);
    }
};
```

二叉树的直径





112、路径总和

```cpp
class Solution {
public:
    bool hasPathSum(TreeNode* root, int targetSum) {
        if (root == nullptr) return false;
        if (targetSum == root->val && !root->left && !root->right) return true;
        return hasPathSum(root->left, targetSum - root->val) || hasPathSum(root->right, targetSum - root->val);
    }
};
```

236、二叉树的最近公共祖先

```cpp
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (root == nullptr) return nullptr;
        if (q == root) return q;
        if (p == root) return p;
        TreeNode *left = lowestCommonAncestor(root->left, p, q);
        TreeNode *right = lowestCommonAncestor(root->right, p, q);
        if (left != nullptr && right == nullptr) return left;
        if (left == nullptr && right != nullptr) return right;
        if (left == nullptr && right == nullptr) return nullptr;
        return root;
    }
};
```



110、平衡二叉树

```cpp
class Solution {
public:
    bool isBalanced(TreeNode* root) {
        if (root == nullptr) return true;
        int left = getHeight(root->left);
        int right = getHeight(root->right);
        if (abs(left - right) <= 1) return isBalanced(root->left) && isBalanced(root->right);
        return false;
    }

    int getHeight(TreeNode * root) {
        if (root == nullptr) return 0;
        return max(getHeight(root->left), getHeight(root->right)) + 1;
    }
};
```

129、求根节点到叶节点数字之和

```cpp
class Solution {
public:
    int sumNumbers(TreeNode* root, int pre = 0) {
        if (root->left == nullptr && root->right == nullptr) return pre * 10 + root->val;
        int left = 0, right = 0;
        if (root->left) left = sumNumbers(root->left, pre * 10 + root->val);
        if (root->right) right = sumNumbers(root->right, pre * 10 + root->val);
        return left + right;
    }
};
```

100、相同的树

```cpp
class Solution {
public:
    bool isSameTree(TreeNode* p, TreeNode* q) {
        if (p == nullptr && q == nullptr) return true;
        if (p == nullptr || q == nullptr) return false;
        if (p->val == q->val) return isSameTree(p->left, q->left) && isSameTree(p->right, q->right);
        return false;
    }
};
```

226、翻转二叉树

```cpp
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        if (root == nullptr) return root;
        swap(root->left, root->right);
        invertTree(root->left);
        invertTree(root->right);
        return root;
    }
};
```

337、打家劫舍

思路：

```cpp
class Solution {
public:
    int rob(TreeNode* root) {
        vector<int> dp = dfs(root);
        return max(dp[0], dp[1]);
    }

    vector<int> dfs(TreeNode* root) {
        if (root == nullptr) return {0, 0};
        vector<int> left = dfs(root->left);
        vector<int> right = dfs(root->right);
        
        int val1 = root->val + left[0] + right[0];
        int val2 = max(left[0], left[1]) + max(right[0], right[1]);
        return {val2, val1};
    }
};
```

106、从中序与后序遍历序列构造二叉树

105、从中序与前序遍历序列构造二叉树

思路：两题思路一致，看下述代码

```cpp
class Solution {
public:
    unordered_map<int, int> index;

    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        int n = inorder.size();
        for (int i = 0; i < n; ++i) {
            index[inorder[i]] = i;
        }
        return myBuildTree(preorder, inorder, 0, preorder.size() - 1, 0, inorder.size() - 1);
    }

    TreeNode* myBuildTree(vector<int>& preorder, vector<int>& inorder, int pl, int pr, int il, int ir) {
        if (il > ir) return nullptr;

        int pre_root = pl;
        int in_root = index[preorder[pl]];

        TreeNode *root = new TreeNode(preorder[pl]);

        int left_size = in_root - il;
        root->left = myBuildTree(preorder, inorder, pl + 1, pl + left_size, il, in_root - 1);
        root->right = myBuildTree(preorder, inorder, pl + left_size + 1, pr, in_root + 1, ir);
        return root;
    }
};
```







## 回溯

### [93. 复原 IP 地址](https://leetcode.cn/problems/restore-ip-addresses/)

思路：

- 递归参数：`void dfs(string s, int start, int num)，num表示在打第num个点，start表示此次选数的起始位置`
- 递归中止条件：`start > s.length(),或者num = 4时；保存结果的条件为num=4且start=s.length()时 `
- 单层搜索的逻辑：搜索1~3长度的字符
- 注意点：
  - 单层搜索时注意`start + i - 1 < s.length()`
  - 判断ip时注意判断02，024之类带前缀0的情况，所以要保证长度在某个范围下再判断合法性
  - 总长度小于4直接返回空即可

```cpp
class Solution {
public:
    vector<string> res;
    string path;

    vector<string> restoreIpAddresses(string s) {
        if (s.length() < 4) return res;
        dfs(s, 0, 0);
        return res;
    }

    bool isIP(string s) {
        int ip = stoi(s);
        if (s.size() == 1) {
            return ip >= 0 && ip <= 9;
        } else if (s.size() == 2) {
            return ip >= 10 && ip <= 99;
        } else {
            return ip >= 100 && ip <= 255;
        }
    }

    void dfs(string s, int start, int num) {
        if (start > s.length()) return;
        if (num == 4) {
            if (start == s.length()) res.push_back(path.substr(0, path.length() - 1));
            return;
        }
        for (int i = 1; i <= 3 && start + i - 1 < s.length(); ++i) {
            string t = s.substr(start, i);
            if (isIP(t)) {
                string path_copy = path;
                path = path + t + '.';
                dfs(s, start + i, num + 1);
                path = path_copy;
            }
        }
    }
};
```



### [22. 括号生成](https://leetcode.cn/problems/generate-parentheses/)

思路：

- 递归参数：`void dfs(int m, int cnt, int left) , m为总长度，cnt为当前长度，left为左括号个数`
- 递归中止条件：`cnt == m时返回，并记录path到res中`
- 单层搜索的逻辑：选择是填左括号还是右括号，左括号必须`left < (m / 2)`，右括号必须保证当前右括号个数小于左括号`cnt - left < left`

```cpp
class Solution {
public:
    vector<string> res;
    string path;

    vector<string> generateParenthesis(int n) {
        dfs(2 * n, 0, 0);
        return res;
    }

    void dfs(int m, int cnt, int left) {
        if (cnt == m) {
            res.push_back(path);
            return;
        }
        if (left < (m / 2)) {
            path.push_back('(');
            dfs(m, cnt + 1, left + 1);
            path.pop_back();
        }
        if (cnt - left < left) {
            path.push_back(')');
            dfs(m, cnt + 1, left);
            path.pop_back();
        }
    }
};
```





## 图论（DFS、BFS、拓扑排序、并查集）



### [200. 岛屿数量](https://leetcode.cn/problems/number-of-islands/)

BFS：

```cpp
class Solution {
public:
    int dx[4] = {0, 0, -1, 1}, dy[4] = {-1, 1, 0, 0};

    int numIslands(vector<vector<char>>& grid) {
        int n = grid.size(), m = grid[0].size();
        vector<vector<bool>> isvisit(n, vector<bool>(m, false));

        int ans = 0;
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (!isvisit[i][j] && grid[i][j] == '1') {
                    bfs(grid, isvisit, i, j);
                    ans++;
                }
            }
        }
        return ans;
    }

    void bfs(vector<vector<char>>& grid, vector<vector<bool>>& isvisit, int i, int j) {
        int n = grid.size(), m = grid[0].size();
        queue<pair<int, int>> q;
        isvisit[i][j] = true;
        q.push({i, j});

        while (!q.empty()) {
            auto t = q.front();
            q.pop();
            int x = t.first, y = t.second;
            for (int i = 0; i < 4; ++i) {
                int nx = dx[i] + x, ny = dy[i] + y;
                if (nx >= 0 && nx < n && ny >= 0 && ny < m && grid[nx][ny] == '1' && !isvisit[nx][ny]) {
                    isvisit[nx][ny] = true;
                    q.push({nx, ny});
                }
            }
        }
    }
};
```



### 循环依赖：DFS、拓扑排序











## 动态规划

### [300. 最长递增子序列](https://leetcode.cn/problems/longest-increasing-subsequence/)

```cpp
class Solution {
public:
    int lengthOfLIS(vector<int>& nums) {
        int n = nums.size();
        if (n == 1) return 1;
        vector<int> dp;
        dp.push_back(nums[0]);

        for (int i = 1; i < n; ++i) {
            if (dp.back() < nums[i]) {
                dp.push_back(nums[i]);
            } else {
                int l = 0, r = dp.size() - 1;
                while (l < r) {
                    int m = l + r >> 1;
                    if (dp[m] >= nums[i]) r = m;
                    else l = m + 1;
                }
                dp[l] = nums[i];
            }
        }
        return dp.size();
    }
};
```



### [5. 最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)

```cpp
class Solution {
public:
    string longestPalindrome(string s) {
        int n = s.length();
        vector<vector<int>> dp(n, vector(n, 0));
        int maxP = 1;
        string res = s.substr(0, 1);
        for (int i = 0; i < n; i++) dp[i][i] = 1;
        for (int i = 0 ; i < n; i++) {
            for (int j = i - 1; j >= 0; j--) {
                if (s[i] == s[j] && dp[i - 1][j + 1] != 0) {
                    dp[i][j] = dp[i - 1][j + 1] + 2;
                }
                if (s[i] == s[j] && i - j == 1) {
                    dp[i][j] = 2;
                }
                if (maxP < dp[i][j]) {
                    maxP = dp[i][j];
                    res = s.substr(j, i - j + 1);
                }
            }
        }
        return res;
    }
};
```

```cpp
class Solution {
public:
    string longestPalindrome(string s) {
        int n = s.length();
        int ans = 0;
        string res;
        for (int i = 0; i < n; ++i) {
            int l = i - 1, r = i + 1;
            while (l >= 0 && r < n && s[l] == s[r]) {
                l--;
                r++;
            }
            if (r - l - 1 > ans) {
                ans = r - l - 1;
                res = s.substr(l + 1, ans);
            }

            l = i, r = i + 1;
            while (l >= 0 && r < n && s[l] == s[r]) {
                l--;
                r++;
            }
            if (r - l - 1 > ans) {
                ans = r - l - 1;
                res = s.substr(l + 1, ans);
            }
        }
        return res;
    }
};
```



### [322. 零钱兑换](https://leetcode.cn/problems/coin-change/)

思路：

- 状态表示：`dp[j]：凑够j块钱需要的最少硬币数`
- 状态转移：`dp[j] = min(dp[j], dp[j - coins[i]] + 1)`
- 初始状态：`dp[0] = 0`
- 此题是一道完全背包型dp

```
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        int n = coins.size();
        vector<int> dp(amount + 1, INT_MAX);
        dp[0] = 0;
        for (int i = 0; i < n; ++i) {
            for (int j = coins[i]; j <= amount; ++j) {
                if (dp[j - coins[i]] != INT_MAX) {
                    dp[j] = min(dp[j - coins[i]] + 1, dp[j]);
                }
            }
        }
        if (dp[amount] == INT_MAX) return -1;
        return dp[amount];
    }
};
```



### [121. 买卖股票的最佳时机](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/)

思路：

- 状态表示：`dp[i][0]:第i天仍然持有时最大利润；dp[i][1]：第i天已经卖出时最大利润`

- 状态转移：
  - `dp[i][0] = max(dp[i - 1][0], -prices[i])`（为什么取max？因为这是确保利润最大，所有取max是在找卖出价格最低的）
  - `dp[i][1] = max(dp[i - 1][1], dp[i - 1][0] + prices[i])`

- 初始状态：`dp[0][0] = -prices[0], dp[0][1] = 0`

```
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int n = prices.size();
        vector<vector<int>> dp(n, vector<int>(2, 0));
        dp[0][0] = -prices[0], dp[0][1] = 0;

        for (int i = 1; i < n; ++i) {
            dp[i][0] = max(dp[i - 1][0], -prices[i]);
            dp[i][1] = max(dp[i - 1][1], dp[i - 1][0] + prices[i]);
        }

        return max(dp[n - 1][0], dp[n - 1][1]);
    }
};
```



### [213. 打家劫舍 II](https://leetcode.cn/problems/house-robber-ii/)





### [494. 目标和](https://leetcode.cn/problems/target-sum/)

思路：

- 状态表示：`dp[i][j]:用前i个物品组合出目标和为j的方法数`
- 状态转移：`dp[i][j] = dp[i - 1][j] + dp[i - 1][j - nums[i]]`
- 初始状态：`dp[i][0] = 1`
- 转换成一维，转移方程为`dp[j] += dp[j - nums[i]]`

```cpp
class Solution {
public:
    int findTargetSumWays(vector<int>& nums, int target) {
        int sum = 0;
        for (auto num : nums) {
            sum += num;
        }
        if (sum < abs(target)){
            return 0;
        }
        if ((sum + target) % 2 == 1) {
            return 0;
        }
        int bg = (sum + target) / 2;
        vector<int> dp(bg + 1, 0);
        dp[0] = 1;
        for (int i = 0; i < nums.size(); ++i) {
            for (int j = bg; j >= nums[i]; --j) {
                dp[j] += dp[j - nums[i]];
            }
        }
        return dp[bg];
    }
};
```



### [72. 编辑距离](https://leetcode.cn/problems/edit-distance/)

思路：

- 状态定义：`dp[i][j]:以word1[i]为尾的串和以word2[j]为尾的串的编辑距离，即相同时的最少操作数`
- 状态转移：
  - `dp[i][j] = min(dp[i-1][j-1], min(dp[i-1][j] + 1, dp[i][j - 1] + 1))[当word1[i-1]等于word2[j-1]]时`
  - `dp[i][j] = min(dp[i-1][j-1] + 1, min(dp[i-1][j] + 1, dp[i][j - 1] + 1))[当word1[i-1]不等于word2[j-1]]时`
- 初始状态：`dp[i][0] = i, dp[0][j] = j`

```cpp
class Solution {
public:
    int minDistance(string word1, string word2) {
        int n = word1.length(), m = word2.length();
        vector<vector<int>> dp(n + 1, vector<int>(m + 1, 0));

        for (int i = 0; i <= n; ++i) {
            dp[i][0] = i;
        }
        for (int j = 0; j <= m; ++j) {
            dp[0][j] = j;
        }

        for (int i = 1; i <= n; ++i) {
            for (int j = 1; j <= m; ++j) {
                if (word1[i - 1] == word2[j - 1]) {
                    dp[i][j] = min(dp[i - 1][j - 1], min(dp[i - 1][j] + 1, dp[i][j - 1] + 1));
                } else {
                    dp[i][j] = min(dp[i - 1][j - 1] + 1, min(dp[i - 1][j] + 1, dp[i][j - 1] + 1));
                }
            }
        }
        return dp[n][m];
    }
};
```







## 贪心







## 数据结构







## 小技巧







## 模拟

### [415. 字符串相加](https://leetcode.cn/problems/add-strings/)

代码简便写法：

```cpp
class Solution {
public:
    string addStrings(string num1, string num2) {
        int i = num1.size() - 1, j = num2.size() - 1;
        string ans;
        int carry = 0;
        while (i >= 0 || j >= 0 || carry != 0) {
            int a = i >= 0 ? num1[i] - '0' : 0;
            int b = j >= 0 ? num2[j] - '0' : 0;
            int sum = a + b + carry;
            ans.push_back((sum % 10) + '0');
            carry = sum / 10;
            i--;
            j--;
        }
        reverse(ans.begin(), ans.end());
        return ans;
    }
};
```







## 面试手撕

### [146. LRU 缓存](https://leetcode.cn/problems/lru-cache/)

思路：双链表+哈希



### 快速排序



### 堆排序



### 归并排序













