# 码上精进算法21天打卡活动·Day 2		
			

## 关于数组基础及二分查找及其知识点 

### 二分搜索算法核心代码模板

框架中两处 ... 表示的更新窗口数据的地方，在具体的题目中，你需要做的就是往这里面填代码逻辑。而且，这两个 ... 处的操作分别是扩大和缩小窗口的更新操作，等会你会发现它们操作是完全对称的。

基于这个框架，遇到子串/子数组相关的题目，你只需要回答以下三个问题：

1、什么时候应该移动 right 扩大窗口？窗口加入字符时，应该更新哪些数据？

2、什么时候窗口应该暂停扩大，开始移动 left 缩小窗口？从窗口移出字符时，应该更新哪些数据？

3、什么时候应该更新结果？

只要能回答这三个问题，就说明可以使用滑动窗口技巧解题。

下面就直接上四道力扣原题来套这个框架，其中第一道题会详细说明其原理，其他题目就直接闭眼睛秒杀了。

```cpp
class Solution {
public:
    int minSubArrayLen(int target, vector<int>& nums) {
        int sum = 0;
        int i = 0;
        int result = INT_MAX;
        for(int j = 0;j < nums.size();j++){
            sum += nums[j];
            while(sum >= target){
                int subL = j - i + 1;
                result  = min(result,subL);
                sum -= nums[i];
                i++;
            }
        }
        return result == INT_MAX ? 0 : result;
    }
};
```