# 算法

## 算法题：

[CodeTop](https://codetop.cc/home)

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

