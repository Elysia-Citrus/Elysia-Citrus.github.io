---
title: <题外话>算法练习 Day 2：滑动窗口、前缀和、模拟
published: 2026-06-11
description: 滑动窗口、螺旋矩阵、前缀和、二维前缀和
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日复盘

主要学习了滑动窗口（双指针的变体）、模拟法（螺旋矩阵）、前缀和及其二维应用。

滑动窗口是双指针的一种特殊形式，通常用于寻找满足条件的连续子数组/子串。核心在于右指针扩展窗口、左指针收缩窗口，两者均单向移动，时间复杂度 O(n)。关键点是：当窗口满足条件时，应先更新结果再收缩窗口，因为当前窗口长度是一个潜在的最优解。

螺旋矩阵属于模拟题，难点在于方向切换的时机判断和边界处理。使用方向数组 + 越界/已访问判断可以简洁地实现转向逻辑。

前缀和是处理区间求和查询的经典预处理技巧，O(n) 预处理后即可 O(1) 查询任意区间和，在多查询场景下相比暴力法有显著优势。推广到二维场景时，可以按行或按列累加，用于解决区域划分问题。

## 滑动窗口：长度最小的子数组

给定一个含有 n 个正整数的数组和一个正整数 target，找出该数组中满足其和 ≥ target 的长度最小的连续子数组，并返回其长度。如果不存在，返回 0。

暴力法需要 O(n²) 枚举所有子数组。滑动窗口可以将复杂度降到 O(n)：右指针 j 遍历数组并累加元素，当窗口和 ≥ target 时，记录当前窗口长度，然后移动左指针 i 收缩窗口（减去 nums[i]），直到窗口和不再满足条件。

关键细节：当窗口和满足条件时，应当先更新结果（`ans = min(ans, j - i + 1)`），再收缩窗口。因为当前窗口长度本身就是一个候选解，如果先收缩再更新，会遗漏这个解。

初始位置始终单向移动，不会回溯，这是滑动窗口与暴力枚举的本质区别。

代码：

```cpp
#include <vector>
#include <climits>
#include <iostream>
using namespace std;

class Solution {
public:
    int minSubArrayLen(int target, vector<int>& nums) {
        int i = 0;
        int ans = INT_MAX;
        int temp = 0;
        int n = nums.size();

        for (int j = 0; j < n; j++) {
            temp += nums[j];

            while (temp >= target) {
                ans = min(ans, j - i + 1);
                temp -= nums[i];
                i++;
            }
        }
        return ans == INT_MAX ? 0 : ans;
    }
};

int main() {
    vector<int> nums = {2, 3, 1, 2, 4, 3};
    int target = 7;
    Solution s;
    int result = s.minSubArrayLen(target, nums);
    cout << "Minimum length of subarray: " << result << endl;
    return 0;
}
```

## 螺旋矩阵

给定一个正整数 n，生成一个包含 1 到 n² 所有元素，且元素按顺时针顺序螺旋排列的 n × n 正方形矩阵。

这道题属于模拟类——不涉及复杂的数据结构或算法技巧，关键在于正确模拟螺旋填充的过程。主要难点：

1. **方向切换**：螺旋填充按照“右→下→左→上”的顺序循环。使用方向数组 `DIRS[4][2]` 定义四个方向的坐标偏移量，通过变量 `di` 控制当前方向，`di = (di + 1) % 4` 实现循环切换。

2. **转向判断**：当下一步越界（`x < 0 || x >= n || y < 0 || y >= n`）或下一步位置已经被填充过（`ans[x][y] != 0`），就需要切换方向。

3. **边界自然收缩**：由于已填充的位置会被跳过，边界会自然地逐层向内收缩，无需手动维护边界变量。

代码：

```cpp
#include <vector>
#include <iostream>
using namespace std;

class Solution {
    static constexpr int DIRS[4][2] = {{0, 1}, {1, 0}, {0, -1}, {-1, 0}};  // 右、下、左、上

public:
    vector<vector<int>> generateMatrix(int n) {
        vector<vector<int>> ans(n, vector<int>(n));
        int i = 0, j = 0, di = 0;

        for (int val = 1; val <= n * n; val++) {
            ans[i][j] = val;

            int x = i + DIRS[di][0];
            int y = j + DIRS[di][1];

            if (x < 0 || x >= n || y < 0 || y >= n || ans[x][y] != 0) {
                di = (di + 1) % 4;
                x = i + DIRS[di][0];
                y = j + DIRS[di][1];
            }

            i = x;
            j = y;
        }
        return ans;
    }
};

int main() {
    int n = 3;
    Solution s;
    vector<vector<int>> result = s.generateMatrix(n);
    cout << "Generated matrix: " << endl;
    for (const auto& row : result) {
        for (int num : row) {
            cout << num << " ";
        }
        cout << endl;
    }
    return 0;
}
```

> **原代码注意点**：方向数组的顺序和初始方向 `di` 需要匹配。如果方向数组定义为 `{上, 下, 左, 右}` 但期望的螺旋顺序是“右、下、左、上”，需要调整数组顺序或初始 `di` 值。此外，计算完下一位置 `(x, y)` 后，别忘了将 `i = x; j = y;` 更新当前位置。

## 前缀和：区间和

给定一个整数数组，多次查询指定区间 `[a, b]` 内元素的总和。

暴力法每次查询需要遍历区间，复杂度 O(n)；如果有 q 次查询，总复杂度为 O(qn)。前缀和的核心思想是预处理一个前缀和数组 `p`，其中 `p[i]` 表示原数组从下标 0 到下标 i 的元素总和。预处理后，任意区间 `[a, b]` 的和可以在 O(1) 时间内计算：

- 若 `a == 0`：`sum = p[b]`
- 若 `a > 0`：`sum = p[b] - p[a - 1]`（注意是 `a - 1`，不是 `a`，因为区间包含 a）

预处理 O(n)，每次查询 O(1)，总复杂度 O(n + q)，远优于暴力的 O(qn)。

代码：

```cpp
#include <vector>
#include <iostream>
using namespace std;

int main() {
    int n, a, b;
    cin >> n;
    vector<int> nums(n);
    vector<int> p(n);      // 前缀和数组
    int presum = 0;

    for (int i = 0; i < n; i++) {
        scanf("%d", &nums[i]);
        presum += nums[i];
        p[i] = presum;
    }

    while (~scanf("%d %d", &a, &b)) {
        int sum;
        if (a == 0)
            sum = p[b];
        else
            sum = p[b] - p[a - 1];
        printf("%d\n", sum);
    }
}
```

> `~scanf` 的写法：`scanf` 返回成功读取的参数个数，读到 EOF 时返回 `-1`（即 `0xFFFFFFFF`），按位取反后为 `0`，循环终止。这是一种常见的竞赛编程写法。

## 前缀和二维应用：开发商购买土地

在一个 n × m 的网格中，每个区块有不同的土地价值。需要将整块区域按横向或纵向划分成两个连续的子区域（每个子区域至少包含一个区块），使得两个子区域的土地总价值之差最小。输出最小差值。

这道题的难点在于如何枚举划分方式并快速计算两个子区域的总价值。思路：

1. 先计算整个区域的总价值 `sum`。
2. **横向划分**：预处理每一行的价值之和 `horizontal[i]`，然后逐行累加，将前 `i + 1` 行作为 A 公司区域，剩余作为 B 公司区域，计算差值并更新最小值。
3. **纵向划分**：同理，预处理每一列的价值之和 `vertical[j]`，逐列累加并更新最小值。
4. 最终结果取横向和纵向划分的最小差值。

本质上是一维前缀和思想在二维场景的推广：将行或列压缩为一个值后，问题退化为在一维数组上寻找最优分割点。

代码：

```cpp
#include <iostream>
#include <vector>
#include <climits>
using namespace std;

int main() {
    int n, m;
    cin >> n >> m;
    vector<vector<int>> grid(n, vector<int>(m));

    int sum = 0;
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < m; j++) {
            cin >> grid[i][j];
            sum += grid[i][j];
        }
    }

    // 统计每一行的总和
    vector<int> horizontal(n, 0);
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < m; j++) {
            horizontal[i] += grid[i][j];
        }
    }

    // 统计每一列的总和
    vector<int> vertical(m, 0);
    for (int j = 0; j < m; j++) {
        for (int i = 0; i < n; i++) {
            vertical[j] += grid[i][j];
        }
    }

    int result = INT_MAX;

    // 横向划分
    int horizontalCut = 0;
    for (int i = 0; i < n - 1; i++) {
        horizontalCut += horizontal[i];
        int difference = abs(sum - 2 * horizontalCut);
        result = min(result, difference);
    }

    // 纵向划分
    int verticalCut = 0;
    for (int j = 0; j < m - 1; j++) {
        verticalCut += vertical[j];
        int difference = abs(sum - 2 * verticalCut);
        result = min(result, difference);
    }

    cout << result << endl;
    return 0;
}
```

> 计算差值时，设 A 公司区域价值为 `cut`，B 公司为 `sum - cut`，差值为 `|cut - (sum - cut)| = |sum - 2 * cut|`。利用这个等式可以简化代码。
