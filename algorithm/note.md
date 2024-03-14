# 算法

[CodeTop](https://codetop.cc/home)

## api
自定义排：
sort.Slice(slice, func(i, j int) bool {

})

正序排：
sort.Ints(slice)


sort.Search
func Search(n int, f func(int) bool) int 
Search uses binary search to find and return the smallest index i
// in [0, n) at which f(i) is true,

移除头尾的某个字符
strings.Trim(str, ch)
str = strings.ToLower(str) 转小写

rand.Intn(n) 随机返回 [0,n)

rand.Float64() 随机返回 [0.0,1.0)


已经开辟好的两个 slice
copy(dst, src)

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

# 数据结构

## 红黑树

自平衡的二叉查找树

| 平均    | 最差 |
| -------- | ------- |
| 空间 O(n)  |     |
| 搜索 O(log n) | O(log n)   |
| 插入 O(1)    | O(log n)    |
| 删除 O(1)   | O(log n)     |

```
性质1：每个节点要么是黑色，要么是红色。
性质2：根节点是黑色。
性质3：每个叶子节点（NIL）是黑色。
性质4：每个红色结点的两个子结点一定都是黑色。
性质5：任意一结点到每个叶子结点的路径都包含数量相同的黑结点。
```

前面讲到红黑树能自平衡，它靠的是什么？三种操作：左旋、右旋和变色。

红黑树相对于AVL树来说，牺牲了部分平衡性以换取插入和删除操作时少量的旋转操作，整体来说性能要优于AVL树。
AVL 树平均的插入和删除都需要 O(log n) 

从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。结果是这个树大致上是平衡的。因为操作比如插入、删除和查找某个值的最坏情况时间都要求与树的高度成比例，这个在高度上的理论上限允许红黑树在最坏情况下都是高效的，而不同于普通的二叉查找树。

要知道为什么这些性质确保了这个结果，注意到性质4导致了路径不能有两个毗连的红色节点就足够了。最短的可能路径都是黑色节点，最长的可能路径有交替的红色和黑色节点。因为根据性质5所有最长的路径都有相同数目的黑色节点，这就表明了没有路径能多于任何其他路径的两倍长。

## 跳跃表
它使得包含n个元素的有序序列的查找和插入操作的平均时间复杂度都是 O(log n)

快速的查询效果是通过维护一个多层次的链表实现的，且与前一层（下面一层）链表元素的数量相比，每一层链表中的元素的数量更少。

一开始时，算法在最稀疏的层次进行搜索，直至需要查找的元素在该层两个相邻的元素中间。这时，算法将跳转到下一个层次，重复刚才的搜索，直到找到需要查找的元素为止。跳过的元素的方法可以是随机性选择[2]或确定性选择[3]，其中前者更为常见。


![图片](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240313-111342.png)


| 平均    | 最差 |
| -------- | ------- |
| 空间 O(n)  |  n*logn   |
| 搜索 log n | n    |
| 插入 log n | n    |
| 删除 log n | n    |


跳跃列表是按层建造的。底层是一个普通的有序链表。每个更高层都充当下面列表的“快速通道”，这里在第 i 层的元素按某个固定的概率（1/2 ，或 1/4）出现在第 i+1 层中。
每个元素平均出现在 1/(1-p) 个列表中

![图片](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240313-112602.png)

每层链表中预期的查找步数最多为 1/p, 层数为 log(1/p)n, 总的步数是 1/p * log(1/p)n

跳跃列表不像平衡树等数据结构那样提供对最坏情况的性能保证：由于用来建造跳跃列表采用随机选取元素进入更高层的方法，在小概率情况下会生成一个不平衡的跳跃列表（最坏情况例如最底层仅有一个元素进入了更高层，此时跳跃列表的查找与普通列表一致）。





