# 码上精进算法21天打卡活动·Day 8		
			

## 关于数组基础及二分查找及其知识点 

### 二分搜索算法核心代码模板






```cpp
class Solution {
public:
    void merge(vector<int>& nums1, int m, vector<int>& nums2, int n) {
        
            int p1 = m - 1;
            int p2 = n - 1;
            int writePos = m + n - 1;
        while (p2 >= 0){
            if (p1 >= 0 && nums1[p1] > nums2[p2]){
                nums1[writePos] = nums1[p1];
                p1 -= 1;
            }else{
                nums1[writePos] = nums2[p2];
                p2 -= 1;
            
            }
            writePos -=1;
        }
    }
};
```