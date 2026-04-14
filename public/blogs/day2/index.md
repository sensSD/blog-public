# 码上精进算法21天打卡活动·Day 2		
			

## 关于数组基础及二分查找及其知识点 

### 二分搜索算法核心代码模板

简单解释一下什么是原地修改：

如果不是原地修改的话，我们直接 new 一个 int[] 数组，把去重之后的元素放进这个新数组中，然后返回这个新数组即可。

但是现在题目让你原地删除，不允许 new 新数组，只能在原数组上操作，然后返回一个长度，这样就可以通过返回的长度和原始数组得到我们去重后的元素有哪些了。

由于数组已经排序，所以重复的元素一定连在一起，找出它们并不难。但如果毎找到一个重复元素就立即原地删除它，由于数组中删除元素涉及数据搬移，整个时间复杂度是会达到 O(N ²)。

高效解决这道题就要用到快慢指针技巧：

我们让慢指针 slow 走在后面，快指针 fast 走在前面探路，找到一个不重复的元素就赋值给 slow 并让 slow 前进一步。

这样，就保证了 nums[0..slow] 都是无重复的元素，当 fast 指针遍历完整个数组 nums 后，nums[0..slow] 就是整个数组去重之后的结果。





```cpp
class Solution {
public:
    int removeElement(vector<int>& nums, int val) {
        return byPointer2(nums, val);
    }

private:
    int byPointer1(vector<int>& nums, int val) {
        int len = nums.size();
        int left = 0;
        for (int right = 0; right < len; ++right) {
            if (nums[right] != val) {
                nums[left++] = nums[right];
            }
        }

        return left;
    }

    int byPointer2(vector<int>& nums, int val) {
        int left = 0;
        int right = nums.size();

        while (left < right) {
            if (nums[left] == val) {
                nums[left] = nums[right - 1];
                right -= 1;
            } else {
                left += 1;
            }
        }

        return left;
    }
};
```