# 码上精进算法21天打卡活动·Day 9	
			

## 关于数组基础及二分查找及其知识点 

### 二分搜索算法核心代码模板

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> numMap; // 存储 数值 -> 索引 的映射
        for (int i = 0; i < nums.size(); ++i) {
            int complement = target - nums[i]; // 计算需要匹配的差值
            if (numMap.count(complement)) {    // 如果差值已在哈希表中
                return {numMap[complement], i};
            }
            numMap[nums[i]] = i;               // 将当前数值和下标存入哈希表
        }
        return {}; // 题目保证有且仅有一组解，此处不会执行
    }
};
```