---
title: <题外话>算法练习 Day 1：数组
published: 2026-06-10
description: 二分法、移除元素、有序数组的平方
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日复盘

主要对数组进行复习，以及学习了二分法和双指针的使用。

二分法的易错点在于选取区间对后续构建边界条件的影响，主流的方法是左闭右闭和左闭右开。当使用左闭右闭时，while条件应当是<=以确保区间合法，更新时也应当是mid ± 1，以筛去已经判定过的下标；当使用左闭右开时，while条件应当是<以确保区间合法，right在更新时也应当是right = mid，以利用开区间的天然筛选性。

双指针能够减少暴力枚举复杂度，当题目描述给定有序数组，且有两数关系时；或是原地修改时；或是最长/最短的连续区间时，就可以尝试使用双指针解决（链表也有对应的情景，届时再更新）。

## 二分查找

二分查找在数据结构这门课中有所涉及，主要用于在有序数组中查找特定元素。它的基本思想是通过不断将搜索范围缩小一半来快速定位目标元素。

在写二分法代码时时常容易写错：

1. while(left < right)以及while(left <= right)的区别。
2. 以及if (nums[mid] < target)和if (nums[mid] <= target)，right = mid - 1和right = mid的区别。

在写二分法时，要么是左闭右闭[right, left]区间，要么是左闭右开[right, left)区间。这两个区间影响边界条件的处理。

在while循环中，应当根据选取区间的类型，选择合适的边界条件。

### 左闭右闭区间[left, right]：

如果使用左闭右闭区间[left, right]，则循环条件应为while (left <= right)。当left == right时，是存在一个合法的区间的，此时mid会等于left和right，仍然需要进行比较。例如[left, right] = [4, 4]，mid = 4.

在写if条件时，如果nums[mid] < target, 说明目标元素在mid右侧，因此left = mid + 1; 如果nums[mid] > target, 说明目标元素在mid左侧，因此right = mid - 1; 如果nums[mid] == target, 则直接返回mid。

right = mid - 1和mid的区别在于，前者会将mid排除在下一轮搜索范围之外，而后者会将mid包含在下一轮搜索范围内。对于左闭右闭区间来说，当nums[mid] > target时，right = mid - 1是正确的，因为nums[mid]已经被比较过了，不需要再包含在下一轮搜索范围内。

对于左闭右闭区间，写mid - 1是合理的，否则可能导致死循环。

### 左闭右开区间[left, right)：

使用左闭右开区间，则循环条件应为while (left < right)。因为[1,1)不能成立一个区间，right一定要大于left。

此时写if条件，如果nums[mid] > target, 下一次搜索的左区间是不包含mid的，那么这时right正好==mid，因为mid以及被比较过了，而我们选取的是开区间，mid不被包含。

若nums[mid] < target, 说明middle不是target，那么下一次的区间不应该包含mid，所以left = mid + 1, 符合左闭原则.

代码：

```cpp
#include <vector>
#include <iostream>
using namespace std;
class Solution {  //左闭右闭区间[left, right]
public:
    int search(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1;

        while (left <= right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] < target) {
                left = mid + 1;
            } else if (nums[mid] > target) {
                right = mid - 1;
            } else {
                return mid; //因为逻辑里无非三种情况：小于、等于、大于，所以当等于时直接返回mid即可。
            }
        }

        return -1;
    }
};

class Solution{  //左闭右开区间[left, right)
public:
    int search(vector<int>&nums, int target){
        int left = 0, right = nums.size();

        while(left < right){
            int mid = left + (right - left) / 2;

            if(nums[mid]< target){
                left = mid + 1;
            }else if(nums[mid] > right){
                right = mid;
            }else{
                return mid;
            }
        }
        return -1;
    }
};


int main() {
    vector<int> nums = {-1, 0, 3, 5, 9, 12};
    int target = 9;

    Solution s;
    int result = s.search(nums, target);

    cout << result << endl;

    return 0;
}
```

## 移除元素

数组的移除元素问题是一个常见的问题，通常在原地修改数组，使得所有不等于指定值的元素都移到数组的前面，并返回新的长度.

这个问题可以通过双指针解决，其中左指针用于跟踪新数组的末尾位置，右指针用于遍历原数组。当右指针遇到不等于指定值的元素时，将其复制到左指针的位置，并将左指针向前移动。最终，左指针的位置即为新数组的长度.

过程是，左指针初始化为0，右指针从0开始遍历数组。当右指针指向的元素不等于val时，将该元素复制到左指针的位置，并将左指针向前移动。这样，所有不等于val的元素都会被移动到数组的前面，而左指针最终的位置就是新数组的长度.

代码：

```cpp
#include <vector>
#include <iostream>
using namespace std;
class Solution {
public:
    int removeElement(vector<int>& nums, int val) {
        int left = 0;

        for(int right = 0; right < nums.size(); right++){
            if(nums[right] != val){
                nums[left] = nums[right];
                left++;
            }
        
        }
        return left;

    }
};


int main(){
    vector<int> nums = {3,2,2,3};
    int val = 3;
    Solution s;
    int newLength = s.removeElement(nums, val);
    cout << "New length: " << newLength << endl;
    cout << "Modified array: ";
    for (int i = 0; i < newLength; i++) {
        cout << nums[i] << " ";
    }
    cout << endl;
    return 0;
}

```

## 双指针案例：有序数组的平方

给定一个按非递减顺序排序的整数数组，返回每个数字的平方组成的新数组，也按非递减顺序排序。

这个问题可以利用的点在于“非递减顺序”的特性，这个特性使得数组两边的绝对值最大。我们可以采用双指针法，
一个指针从数组的左端开始，另一个指针从数组的右端开始。比较两个指针所指向的元素的绝对值，较大的那个元素的平方将被放入结果数组的末尾。然后移动相应的指针，继续比较，直到两个指针相遇。

在新数组写入时，应当从右边写入，因为我们是从两端比较绝对值，较大的平方应该放在结果数组的末尾。控制写入指针p不断-1，左指针i不断+1，右指针j不断-1，直到i和j相遇。

代码：

```cpp
#include <vector>
#include <iostream>
using namespace std;

class Solution {
public:
    vector<int> sortedSquares(vector<int>& nums) {
        int n = nums.size();  //长度
        vector<int> ans(n); //答案

        int i = 0; //左指针
        int j = n - 1; //右指针
        int p = n - 1; //写入指针

        while(i <= j){
            int i_num = nums[i] * nums[i];
            int j_num = nums[j] * nums[j];

            if(i_num < j_num){
                ans[p] = j_num;
                j--;
                p--;
            }else{
                ans[p] = i_num;
                i++;
                p--;
            }
        }
        return ans;
    }
};

int main(){
    vector<int> nums = {-4,-1,0,3,10};
    Solution s;
    vector<int> result = s.sortedSquares(nums);
    cout << "Sorted squares: ";
    for (int num : result) {
        cout << num << " ";
    }
    cout << endl;
    return 0;
}

```