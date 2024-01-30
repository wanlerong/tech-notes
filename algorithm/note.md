# 算法

## 算法题：

[CodeTop](https://codetop.cc/home)

## 总结


### 链表：
思路：多指定几个 p0, p1, p2, p3

LRU 一个双向链表 + map
LFU map[fre]双向链表 + map

K 个一组翻转链表: 节点总数/k 需要反转多少轮；再逐轮反转

合并两个有序链表：直接递归


是否是环形链表：快慢指针，只要能相遇就是环形链表。
返回链表开始入环的第一个节点： 快慢指针转2圈得到环的长度，第二次相遇步数 - 第一次相遇步数；然后用两个等速指针，一个先走环的长度
删除链表的倒数第N个节点：先走N步，再等速指针

相交链表：找链表的焦点。两个等速指针，走完后切到对方的head上。（指针走的距离的利用，等速指针还可以某一个先走一段距离）

合并k个排序链表：不断砍半递归（递归到两个 len 1 的链表arr 合并为止），最后合并两个一半长的。

重排链表：可以用数组存链表的每个节点

删除排序链表中的重复元素 II：一次遍历

排序链表，自底向上，快慢指针找中点，再不停对半拆开


### 滑动窗口：
无重复字符的「最长子串」


### 堆
求第k个最大的，topK

滑动窗口最大值： 优先队列，push 下标


### 双指针

三树之和：先排序 + 双指针

接雨水

### 前缀和

最大子数组和

### hashmap

两数之和

### 动态规划

最长回文子串
if dp[i+1][j-1] && s[i] == s[j] {
    dp[i][j] = true
}

编辑距离

op[i][j] 表示 A 的前 i 个字母和 B 的前 j 个字母之间的编辑距离。

if word1[i-1] == word2[j-1] {
    op[i][j] = op[i-1][j-1]
} else {
    op[i][j] = 1 + min(min(op[i-1][j], op[i-1][j-1]), op[i][j-1])
}

最长公共子序列

i，j 表示两个字符串的前缀长度

if text1[i] == text2[j] {
    dp[i+1][j+1] = dp[i][j] + 1
} else {
    dp[i+1][j+1] = max(dp[i+1][j], dp[i][j+1])
}

最长递增子序列
也可以贪心 + 二分


零钱兑换：

即我们枚举最后一枚硬币面额是

  if (coins[j] <= i) {
                    dp[i] = min(dp[i], dp[i - coins[j]] + 1);
                }

​

定义状态 dp[i][0]\textit{dp}[i][0]dp[i][0] 表示第 iii 天交易完后手里没有股票的最大利润，dp[i][1]\textit{dp}[i][1]dp[i][1] 表示第 iii 天交易完后手里持有一支股票的最大利润（iii 从 000 开始）。


最大正方形：右下角
dp[i][j] = min(dp[i][j-1], min(dp[i-1][j-1], dp[i-1][j])) + 1

## 二叉树

二叉树层序遍历：队列 或者 存下每一层的 arr
二叉树的最近公共祖先：直接递归

二叉树中的最大路径和: 递归得到左子树和右子树的贡献，然后 把该点作为子节点的 gain := node.Val + max(left, right)，负贡献置为 0

二叉树的右视图： dfs，优先右侧，记下每一层是否已经添加过。

从前序与中序遍历序列构造二叉树：中序遍历的值为分界点（左右的），递归



## 二分法
在于不断缩小范围，[l,r] 这个范围是左闭右闭的
并且，对于最后一次循环，初始结构都会是 l=m， r=l+1。
因为进循环的条件是, 所以可以根据最后的一次界限变更得到最终的范围，一般都会是 l==r， 或者 l > r 然后break。break 后是最终的界限。

```
for l < r {
    ...
}
```


搜索滚动过的数组


## 图论

岛屿数量 dfs 上下左右都走，先污染后治理。只走一次，记录访问过的节点


## 栈

有效的括号

## 贪心

股票的最佳时机：记下当前（最小值），每次都卖的话的利润的（最大值）
最长递增子序列：贪心 + 二分


## 回溯

全排列：回溯 纵向递归，横向遍历

复原ip地址：纵向递归，横向遍历

括号生成


## 矩阵
螺旋矩阵：round 一圈一圈打印，每圈四条边。


## 用阿斯克码计算

字符串相加


## 其他

合并区间：先按左端点排序，sort.slice
下一个排列：从后方两层扫描 + reverse 某个范围，最终返回的是 reverse


## 快速排序

```
import "math/rand"
import "time"

func sortArray(nums []int) []int {
    rand.Seed(time.Now().UnixNano())
    randSortArray(nums, 0, len(nums) - 1)
    return nums
}

func randSortArray(nums []int, l,r int) {

    if l >= r {
        return
    }

    i,j := l - 1,r + 1

    s := (rand.Intn(100) % (r - l + 1)) + l
    nums[l],nums[s] = nums[s],nums[l]

    p := nums[l]

    for i < j {
        for i++; nums[i] < p; {
            i++
        }
        for j--; nums[j] > p; {
            j--
        }
        if i < j {
            nums[i],nums[j] = nums[j],nums[i]
        }
    }

    randSortArray(nums, l, j)
    randSortArray(nums, j + 1, r)
}
```

## 堆排序

```

func sortArray(nums []int) []int {
    res := []int{}
    heapfy(nums, len(nums))
    
    for {
        sz := len(nums)
        res = append(res, nums[0])
        nums[0],nums[sz-1] = nums[sz-1],nums[0]
        nums = nums[:sz-1]
        if (sz <= 1) {
            break
        }
        heapfy2(nums, sz-1, 0)
    }

    return res
}

func heapfy(nums []int, size int) {
    for i:=size/2; i>=0; i-- {
        heapfy2(nums, size, i)
    }
}

func heapfy2(nums []int, size int, idx int) {
    l,r := 2*idx+1, 2*idx+2

    min := nums[idx]
    minIdx := idx

    if l < size && nums[l] < min {
        min = nums[l]
        minIdx = l
    }
    if r < size && nums[r] < min {
        min = nums[r]
        minIdx = r
    }

    if minIdx == idx {
        return
    } else {
        nums[idx],nums[minIdx] = nums[minIdx],nums[idx]
        heapfy2(nums, size, minIdx)
    }
}

```


