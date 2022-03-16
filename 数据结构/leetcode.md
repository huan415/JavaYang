# leetcode

#### [98. 验证二叉搜索树](https://leetcode-cn.com/problems/validate-binary-search-tree/) 递归，左右子树分别是二叉搜索树

#### [101. 对称二叉树](https://leetcode-cn.com/problems/symmetric-tree/)  递归，左右子树分别对称

#### [102. 二叉树的层序遍历](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/)  广度优先搜索。Deque, 取出一个，就将其左右子树压对列，一层层遍历。

#### [104. 二叉树的最大深度](https://leetcode-cn.com/problems/maximum-depth-of-binary-tree/)  递归，Math.max(左子树，右子树)+1

#### [105. 从前序与中序遍历序列构造二叉树](https://leetcode-cn.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)  递归，前序第一个是根节点，拿根节点将中序一分为二（左右子树）

#### [114. 二叉树展开为链表](https://leetcode-cn.com/problems/flatten-binary-tree-to-linked-list/)  cur.left!=null 将cur.next赋值给cur.right, 并找一个地方放置cur.right

#### [121. 买卖股票的最佳时机](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock/) 遍历数组，1. 如果 price< min, 则替换 min;  2. 如果 price> min，则Max(ans,(price-min)

#### [124. 二叉树中的最大路径和](https://leetcode-cn.com/problems/binary-tree-maximum-path-sum/) 递归，当前节点 =  node.val + dfs(node.left) + dfs(node.right)；

#### [128. 最长连续序列](https://leetcode-cn.com/problems/longest-consecutive-sequence/) 先放入set 去重，再遍历数组，!sets.contains(n-1) 说明当前元素是起点，再while一个个遍历sets.contains(n+1)

#### [139. 单词拆分](https://leetcode-cn.com/problems/word-break/) 动态规划，先将字典放入set，遍历s(下标是i)，以j为分隔符，dp[i] = dp[j] && s.substring(j,i)

#### [141. 环形链表](https://leetcode-cn.com/problems/linked-list-cycle/) 快慢指针，while(slow != fast) 直到 if(fast == null || fast.next == null) return false;  否则 return true

#### [142. 环形链表 II](https://leetcode-cn.com/problems/linked-list-cycle-ii/) 快慢指针，第一步：while 循环看有没有环，第二步：重置fast=head, while(fast != slow)就一直走，直到相等时

#### [146. LRU 缓存机制](https://leetcode-cn.com/problems/lru-cache/) 两种方法，方法一：继承 LinkedHashMap<Integer, Integer> 并实现 removeEldestEntry；方法二：	；

#### [152. 乘积最大子数组](https://leetcode-cn.com/problems/maximum-product-subarray/)  三种情况：min{i-1}\*nums[i]、max{i-1}\*nums[i]、nums[i]

#### [155. 最小栈](https://leetcode-cn.com/problems/min-stack/) 两个栈：数据栈（push、pop、top） + 辅助栈(getMin)

#### [160. 相交链表](https://leetcode-cn.com/problems/intersection-of-two-linked-lists/) 双指针，  while (pA != pB) {pA = pA == null ? headB : pA.next;  pB = pB == null ? headA : pB.next;}

#### [169. 多数元素](https://leetcode-cn.com/problems/majority-element/) map 统计各个元素个数，再遍历map查最大

#### [198. 打家劫舍](https://leetcode-cn.com/problems/house-robber/) 动态规划，dp[i] = Math.max(dp[i-2]+nums[i],dp[i-1]); 

#### [200. 岛屿数量](https://leetcode-cn.com/problems/number-of-islands/) 深度优先搜索, 遍历网格，找到一个为‘1’的格子，ans++；并该格子及其四周的各自置为0，继续找其他的。

#### [206. 反转链表](https://leetcode-cn.com/problems/reverse-linked-list/) 迭代反转，reverseList(head.next)；head.next.next = head;head.next = null;return newHead;

#### [207. 课程表](https://leetcode-cn.com/problems/course-schedule/)  深度优先搜索，建议先修和后修的对应关系，并且记录有没有被访问过，然后遍历所有科目

#### [208. 实现 Trie (前缀树)](https://leetcode-cn.com/problems/implement-trie-prefix-tree/) Trie[] children;boolean isEnd;   26个字母，每个字母的位置。

#### [221. 最大正方形](https://leetcode-cn.com/problems/maximal-square/)  dp\[i][j] = Math.min(dp\[i-1][j-1],Math.min(dp\[i-1][j],dp\[i][j-1]))+1;

#### [226. 翻转二叉树](https://leetcode-cn.com/problems/invert-binary-tree/)  递归，root.left = right;  root.right = left;

#### [234. 回文链表](https://leetcode-cn.com/problems/palindrome-linked-list/) 递归，

#### [236. 二叉树的最近公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)  递归，(left && right) || ((root.val == p.val || root.val == q.val) && (left || right))

#### [238. 除自身以外数组的乘积](https://leetcode-cn.com/problems/product-of-array-except-self/)  算两个东西，1：leftSumArray[i] 即：i-1 之前的乘机 2：rightSum  右边的乘机

#### [240. 搜索二维矩阵 II](https://leetcode-cn.com/problems/search-a-2d-matrix-ii/)  Z 字形查找(题目有序)，int i = 0, j = columns-1; while(i<rows && j>=0){if(matrix[i][j] == target) return true;if(matrix[i][j] > target){--j;}else{ ++i;}}

#### [279. 完全平方数](https://leetcode-cn.com/problems/perfect-squares/)  动态规划，dp[i] = Math.min(ans,dp[i-j^2])

#### [283. 移动零](https://leetcode-cn.com/problems/move-zeroes/)  双指针，左指针==》0；右指针==》非0；

#### [287. 寻找重复数](https://leetcode-cn.com/problems/find-the-duplicate-number/)  二分查找法，mid = (l + r) >> 1; 遍历统计<=mid 的个数count，个数count<=min 在右边，否则在左边

#### [300. 最长递增子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)  动态规划，*dp*[*i*]=max(*dp*[*j*])+1,其中0≤*j*<*i*且*num*[*j*]<*num*[*i*]

#### [309. 最佳买卖股票时机含冷冻期](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)  f\[i][0]: 手上持有股票的最大收益
        // f\[i][1]: 手上不持有股票，并且处于冷冻期中的累计最大收益
        // f\[i][2]: 手上不持有股票，并且不在冷冻期中的累计最大收益

#### [322. 零钱兑换](https://leetcode-cn.com/problems/coin-change/)  动态规划,dp[i]:组成金额 i*i* 所需最少的硬币数量。动态转移方程： dp[i] = Math.min(dp[i], dp[i - coins[j]] + 1);

#### [337. 打家劫舍 III](https://leetcode-cn.com/problems/house-robber-iii/)  递归+动态规划 dfs(TreeNode node){ return  new int[]{selected, noSelected};}

#### [338. 比特位计数](https://leetcode-cn.com/problems/counting-bits/)  动态规划——最低设置位   dp[i] = dp[i&(i-1)] + 1;

#### [347. 前 K 个高频元素](https://leetcode-cn.com/problems/top-k-frequent-elements/)  分为下面几步：1. Map 记录数值-频率的关系，2.PriorityQueue 遍历map,将其频率前k的放入队列（队列栈顶是最小值）

#### [394. 字符串解码](https://leetcode-cn.com/problems/decode-string/)  栈（LinkedList），1. 数字，入栈(不止一位)，2. 字母或左括号，入栈，3. 右括号开始出栈，直至出现左括号

#### [406. 根据身高重建队列](https://leetcode-cn.com/problems/queue-reconstruction-by-height/)  1. Arrays.sort 排序，身高不一样，按身高倒序，身高一样，按前面的人数升序。2. 遍历排序后的数组，ans.add(person[1], person);

#### [416. 分割等和子集](https://leetcode-cn.com/problems/partition-equal-subset-sum/)  动态规划，前提判断（先过滤几种情况）：1.数组小于2，2. 数组和不是偶数，3. 最大值>和的一半。转移方程：*dp*[*j*]=*dp*[*j*] ∣ *dp*[*j*−*nums*[*i*]]。

#### [437. 路径总和 III](https://leetcode-cn.com/problems/path-sum-iii/)  解题思路：[前缀和，递归，回溯](https://leetcode-cn.com/problems/path-sum-iii/solution/qian-zhui-he-di-gui-hui-su-by-shi-huo-de-xia-tian/)

#### [438. 找到字符串中所有字母异位词](https://leetcode-cn.com/problems/find-all-anagrams-in-a-string/) 滑动窗口，sCounts(是的滑动窗口)、pCounts(标准值)，遍历字符串并滑动窗口，再比较Arrays.equals(sCounts,pCounts)

#### [448. 找到所有数组中消失的数字](https://leetcode-cn.com/problems/find-all-numbers-disappeared-in-an-array/)  原地修改，第一次遍历：nums[(num-1)%len] +=len;  第二次遍历：if(nums[i] <= len) ans.add(i+1);

#### [461. 汉明距离](https://leetcode-cn.com/problems/hamming-distance/)  Integer.bitCount(x ^ y);

#### [494. 目标和](https://leetcode-cn.com/problems/target-sum/)  动态规划

#### [538. 把二叉搜索树转换为累加树](https://leetcode-cn.com/problems/convert-bst-to-greater-tree/) 递归；三步：1. 递归右子树(题目要求每个节点 = 原树中每个节点的值之和)，2. 更改当前节点，3. 递归左子树

#### [543. 二叉树的直径](https://leetcode-cn.com/problems/diameter-of-binary-tree/)  深度优先搜索，递归，左子树 + 右子树 + 1；

#### [560. 和为 K 的子数组](https://leetcode-cn.com/problems/subarray-sum-equals-k/)  前缀和 + 哈希表优化，*pre*[*j*−1]==*pre*[*i*]−*k*, 看map里面有没有*pre*[*j*−1]的值

#### [581. 最短无序连续子数组](https://leetcode-cn.com/problems/shortest-unsorted-continuous-subarray/)  解题思路：[Java 双指针 代码简洁易懂 思路清晰](https://leetcode-cn.com/problems/shortest-unsorted-continuous-subarray/)

#### [581. 最短无序连续子数组](https://leetcode-cn.com/problems/shortest-unsorted-continuous-subarray/)  双指针，每次更新边界值

#### [617. 合并二叉树](https://leetcode-cn.com/problems/merge-two-binary-trees/)  深度优先搜索，1. newNode = new TreeNode(root1.val + root2.val);  2. 递归左子树、右子树

#### [621. 任务调度器](https://leetcode-cn.com/problems/task-scheduler/)  解题：[任务调度器](https://leetcode-cn.com/problems/task-scheduler/solution/621-ren-wu-diao-du-qi-by-chen-wei-f-ua69/)  1. int[26] 记录每个字母出现的次数，2. 找出字母次数最多，3. times= (maxChar-1) * (n+1)，4. times + 频率最多字母的次数 5. 与任务length 比最大值

#### [647. 回文子串](https://leetcode-cn.com/problems/palindromic-substrings/)  中心拓展

#### [739. 每日温度](https://leetcode-cn.com/problems/daily-temperatures/)  单调栈(存储下标)，1. 遍历数组栈为空或小于栈顶元素，则入账，2. 大于栈顶元素，则出栈，求出