# 码上精进算法21天打卡活动·Day 4		
			

## 关于数组基础及二分查找及其知识点 

### 二分搜索算法核心代码模板

定义四个边界：top（上）、bottom（下）、left（左）、right（右），初始时分别指向矩阵的四边。
按顺时针方向逐圈填充：
上边：left → right，填完后 top++（上边界向内收缩）
右边：top → bottom，填完后 right--
下边：right → left，填完后 bottom--
左边：bottom → top，填完后 left++
防越界处理：在填充下边和左边前加入 if (top <= bottom) 和 if (left <= right) 判断，避免在奇数 n 的中心点重复覆盖或越界访问。
循环终止条件：使用 num <= n * n 精确控制填充次数，自然结束循环。


```cpp
class Solution {
public:
    vector<vector<int>> generateMatrix(int n) {
        vector<vector<int>> matrix(n, vector<int>(n));
        int top = 0, bottom = n - 1;
        int left = 0, right = n - 1;
        int num = 1;
        int target = n * n;

        while (num <= target) {
            // 1. 从左到右填充上边界
            for (int i = left; i <= right; ++i) {
                matrix[top][i] = num++;
            }
            top++;

            // 2. 从上到下填充右边界
            for (int i = top; i <= bottom; ++i) {
                matrix[i][right] = num++;
            }
            right--;

            // 3. 从右到左填充下边界（防单行重复覆盖）
            if (top <= bottom) {
                for (int i = right; i >= left; --i) {
                    matrix[bottom][i] = num++;
                }
                bottom--;
            }

            // 4. 从下到上填充左边界（防单列重复覆盖）
            if (left <= right) {
                for (int i = bottom; i >= top; --i) {
                    matrix[i][left] = num++;
                }
                left++;
            }
        }
        
        return matrix;
    }
};