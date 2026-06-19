---
title: <题外话>算法练习 Day 5：哈希表
published: 2026-06-19
description: 哈希表基础语法、有效字母异位词、赎金信、两个数组的交集、两数之和、四数相加、三数之和、四数之和
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日头条

哈希表的几道题：有效字母异位词、赎金信、两个数组的交集、两数之和、四数相加 II、三数之和、四数之和。

哈希表的核心价值在于快速判断一个元素是否出现过。当题目需要查询某个元素是否在集合里、统计频率、建立映射关系时，应第一时间想到哈希表。

C++ 标准库中常用的哈希结构有三种：

| 结构 | 用途 | 底层 |
|------|------|------|
| `std::unordered_set` | 只关心 key 是否存在，不关心次数 | 哈希表 |
| `std::unordered_map` | key → value 映射，如统计频率、记录下标 | 哈希表 |
| `std::array` / `std::vector` | 当 key 范围有限且连续时（如小写字母 a-z），数组比哈希表更轻量 | 连续内存 |

对于三数之和与四数之和，虽然可以用哈希表做，但去重非常麻烦。排序 + 双指针是更省流的做法——排序后数组有序，可以通过控制指针移动方向来逼近 target，同时跳过重复值。这两道题本质上是双指针题。

## 哈希表语法回顾

### unordered_set —— 判断“出现过没有”

适用场景：去重、检查重复元素、记录访问过的节点/地址/状态。

```cpp
unordered_set<int> st;

st.insert(3);
st.insert(5);

// 判断是否存在：count() 返回 0 或 1
if (st.count(3)) {
    cout << "3 存在" << endl;
}

// find() 返回迭代器，没找到返回 st.end()
if (st.find(5) != st.end()) {
    cout << "5 存在" << endl;
}
```

### unordered_map —— 存储“key → value”关系

适用场景：统计频率、记录某个值对应的下标、记录前缀和出现次数、建立映射关系。

```cpp
unordered_map<int, int> mp;

mp[10] = 3;     // key=10, value=3
mp[20] = 5;     // key=20, value=5

cout << mp[10]; // 输出 3
```

**统计频率**的惯用写法：

```cpp
unordered_map<int, int> cnt;
for (int x : nums) {
    cnt[x]++;   // 如果 cnt[x] 不存在，C++ 会自动创建并初始化为 0
}
```

**记录下标**的惯用写法：

```cpp
unordered_map<int, int> mp;
for (int i = 0; i < nums.size(); i++) {
    mp[nums[i]] = i;  // 数值 → 下标
}
```

**判断 key 是否存在**，推荐两种写法：

```cpp
// 写法一：count() 安全，不会意外插入 key
if (mp.count(key)) { ... }

// 写法二：find() 可以顺带拿到 value
auto it = mp.find(key);
if (it != mp.end()) {
    cout << it->first << " → " << it->second << endl;
}
```

> **注意**：`mp[key]` 即使 key 不存在也会自动插入并初始化为 0。因此判断是否存在时务必用 `count()` 或 `find()`，避免无意中往哈希表里塞入垃圾数据。

**遍历**的惯用写法：

```cpp
unordered_map<int, int> mp;
for (auto &p : mp) {        // 引用写法，避免拷贝
    cout << p.first << " " << p.second << endl;
}
```

`unordered_map` 的每一项是 `pair<const Key, Value>`，其中 `p.first` 是 key，`p.second` 是 value。因此 `auto it = mp.find(target)` 找到后，`it->first` 是 key，`it->second` 是 value。

### 什么时候用数组代替哈希表

当 key 的取值范围有限且连续时，直接用数组更轻量。例如小写字母只有 26 个：

```cpp
int cnt[26] = {0};          // 比 unordered_map<char, int> 更快
cnt[ch - 'a']++;            // 将字符映射到 0~25
```

同理，当题目明确数值范围在 `[0, 1000]` 之类的小区间时，数组是更优选择。

---

## 有效字母异位词（LeetCode 242）

题目：判断字符串 `t` 是否是字符串 `s` 的字母异位词（即两个字符串由相同的字母构成，只是顺序不同）。

### 思路

用一个长度为 26 的数组（或哈希表）统计 `s` 中每个字符的出现次数，然后遍历 `t` 对计数做减法。如果某个字符的计数减到负数，说明 `t` 比 `s` 多用了这个字符，直接返回 false。

C++ 中字符本质上可以参与整数运算：`'b' - 'a' == 1`，因此可以将字母映射到数组下标。

### 关键细节

- 先判断两个字符串长度是否相等，不相等直接返回 false，省去后续计算。
- 用数组而非哈希表：只有 26 个小写字母，`int cnt[26]` 比 `unordered_map` 更轻量。但代码中使用的 `unordered_map` 写法同样正确，只是空间常数稍大。

### 代码

```cpp
bool isAnagram(string s, string t) {
    if (s.size() != t.size()) {
        return false;
    }

    unordered_map<char, int> cnt;
    for (char ch : s) {
        cnt[ch]++;
    }
    for (char ch : t) {
        cnt[ch]--;
        if (cnt[ch] < 0) {
            return false;
        }
    }
    return true;
}
```

> 数组写法（更优）：`int cnt[26] = {0};` 然后 `cnt[ch - 'a']++` / `cnt[ch - 'a']--`。

---

## 赎金信（LeetCode 383）

题目：判断 `ransomNote` 能否由 `magazine` 中的字符构成，每个字符只能使用一次。

### 思路

和有效字母异位词几乎一样：遍历 `magazine` 统计字符频率，再遍历 `ransomNote` 消耗频率。如果某个字符不够用（计数变为负数），返回 false。

### 与有效字母异位词的区别

- 异位词要求两个字符串长度相等且字符组成完全相同。
- 赎金信只要求 `magazine` 能覆盖 `ransomNote`，`magazine` 可以有多余字符。因此不需要先判断长度相等。

### 代码

```cpp
bool canConstruct(string ransomNote, string magazine) {
    unordered_map<char, int> mp;
    for (char ch : magazine) {
        mp[ch]++;
    }
    for (char ch : ransomNote) {
        mp[ch]--;
        if (mp[ch] < 0) {
            return false;
        }
    }
    return true;
}
```

---

## 两个数组的交集（LeetCode 349）

题目：求两个数组的交集，结果中的每个元素必须唯一。

### 思路

利用 `unordered_set` 自动去重的特性：

1. 将 `nums1` 全部塞入 `set1`（自动去重）。
2. 遍历 `nums2`，如果元素在 `set1` 中存在，就插入 `set2`。
3. 将 `set2` 转为 `vector` 返回。

### 关于 set 的返回值

`set.insert(value)` 返回一个 `pair<iterator, bool>`：
- `result.second == true`：插入成功（元素之前不存在）。
- `result.second == false`：插入失败（元素已存在）。

本题不需要用到这个特性，但它是 set 的一个重要细节。

### 代码

```cpp
vector<int> intersection(vector<int>& nums1, vector<int>& nums2) {
    unordered_set<int> set1(nums1.begin(), nums1.end());
    unordered_set<int> set2;

    for (int value : nums2) {
        if (set1.count(value)) {
            set2.insert(value);
        }
    }
    return vector<int>(set2.begin(), set2.end());
}
```

---

## 两数之和（LeetCode 1）

题目：在数组中找到两个数，使它们的和等于 `target`，返回这两个数的下标。假设每种输入只会对应一个答案，且同一个元素不能使用两次。

### 思路

这是哈希表的经典应用。核心思路：遍历数组时，对于当前元素 `nums[i]`，去哈希表中查找 `target - nums[i]` 是否已经出现过。

具体做法（两遍循环版本）：
1. 第一遍：建立 `值 → 下标` 的映射。
2. 第二遍：对于每个 `nums[i]`，查找 `target - nums[i]` 是否存在，且下标不同于 `i`。

### 一遍循环优化

可以合并为一遍循环：遍历时先查表，再把当前元素插入表。这样天然保证了不会用到同一个元素（因为当前元素还没入表）。

```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> mp;
    for (int i = 0; i < nums.size(); i++) {
        int complement = target - nums[i];
        if (mp.count(complement)) {
            return {mp[complement], i};
        }
        mp[nums[i]] = i;
    }
    return {};
}
```

### 两遍循环版本

```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> matched;
    for (int i = 0; i < nums.size(); i++) {
        matched[nums[i]] = i;
    }
    for (int i = 0; i < nums.size(); i++) {
        int complement = target - nums[i];
        if (matched.find(complement) != matched.end()
            && matched[complement] != i) {
            return {i, matched[complement]};
        }
    }
    return {};
}
```

### 关键细节

- 必须判断 `matched[complement] != i`，确保不是同一个元素用了两次。例如 `nums = [3, 2, 4], target = 6`，如果忘记这个判断，`3 + 3 = 6` 会错误返回 `[0, 0]`。

---

## 四数相加 II（LeetCode 454）

题目：给定四个长度相同的数组 `nums1, nums2, nums3, nums4`，计算有多少个元组 `(i, j, k, l)` 使得 `nums1[i] + nums2[j] + nums3[k] + nums4[l] == 0`。

### 思路

分治 + 哈希表。如果直接四重循环，复杂度是 O(n⁴)，不可接受。

将问题拆成两半：
1. 先遍历 `nums1` 和 `nums2`，用哈希表统计所有 `a + b` 的和及其出现次数。
2. 再遍历 `nums3` 和 `nums4`，对于每个 `c + d`，去哈希表中查找 `-(c + d)` 出现的次数，累加到答案中。

复杂度降为 O(n²)。

### 与四数之和（LeetCode 18）的区别

- 四数相加 II：四个独立的数组，统计满足条件的**元组个数**，不要求去重。
- 四数之和：一个数组，找到所有**不重复的四元组**，需要去重。

前者用哈希表分治，后者用排序 + 双指针。

### 代码

```cpp
int fourSumCount(vector<int>& nums1, vector<int>& nums2,
                 vector<int>& nums3, vector<int>& nums4) {
    unordered_map<int, int> cnt;

    // 统计前两个数组中所有两数之和的出现次数
    for (int a : nums1) {
        for (int b : nums2) {
            cnt[a + b]++;
        }
    }

    int ans = 0;
    // 在后两个数组中查找匹配项
    for (int c : nums3) {
        for (int d : nums4) {
            int target = -(c + d);
            auto it = cnt.find(target);
            if (it != cnt.end()) {
                ans += it->second;  // 累加所有匹配的组合数
            }
        }
    }
    return ans;
}
```

### 关键细节

- 使用 `cnt.find(target)` 而非 `cnt[target]`：后者在 key 不存在时会插入一个默认值 0，造成不必要的哈希表膨胀。
- 答案累加 `it->second` 而非 `+1`：因为前两个数组可能有多种组合得到相同的和，每一种都要计入。

---

## 三数之和（LeetCode 15）

题目：在一个数组中找到所有不重复的三元组 `[a, b, c]`，使得 `a + b + c == 0`。

### 为什么不用哈希表

三数之和如果用哈希表做，去重会非常复杂。因为结果要求三元组不能重复（如 `[-1, 0, 1]` 和 `[0, -1, 1]` 算同一个），而哈希表不关心顺序，去重需要额外的逻辑。

### 排序 + 双指针

核心思路：
1. 将数组**排序**（这是双指针法的前提）。
2. 固定第一个数 `nums[i]`，问题退化为在 `[i+1, n-1]` 区间内找两数之和等于 `-nums[i]`。
3. 用双指针 `left` 和 `right` 在排序数组中收窄：
   - 若 `nums[left] + nums[right] < -nums[i]` → `left++`（需要更大的值）。
   - 若 `nums[left] + nums[right] > -nums[i]` → `right--`（需要更小的值）。
   - 若相等 → 记录结果，然后同时移动 `left` 和 `right`。

排序的重要性：排序后，数组从左到右递增。双指针可以根据当前和与 target 的大小关系，有方向地移动，不会遗漏。

### 去重逻辑（最容易出错的地方）

**对 a（第一个数）去重**：

```cpp
if (i > 0 && nums[i] == nums[i - 1]) {
    continue;  // 跳过重复的 a
}
```

> **容易写错的地方**：写成 `nums[i] == nums[i + 1]` 会出错。`nums[i] == nums[i - 1]` 才是对结果集去重的正确写法。因为 `nums[i] == nums[i + 1]` 判断的是三元组内部是否有重复元素，而 `[-1, -1, 2]` 这样的答案中元素是可以重复的。真正要避免的是结果集中出现两个相同的三元组，即前一个 a 已经搜过的值不要再搜第二遍。

**对 b 和 c 去重**：找到一个答案后，跳过相邻的重复值。

```cpp
while (left < right && nums[right] == nums[right - 1]) right--;
while (left < right && nums[left] == nums[left + 1]) left++;
right--;
left++;
```

> 注意：`left++` 而非 `left--`。双指针需要向中间收窄，所以 left 向右移，right 向左移。

### 剪枝

当 `nums[i] > 0` 时可以直接 break。因为数组已排序，第一个数大于 0 意味着后面所有数都 ≥ 0，三数之和不可能等于 0。

### 代码

```cpp
vector<vector<int>> threeSum(vector<int>& nums) {
    vector<vector<int>> result;
    sort(nums.begin(), nums.end());

    for (int i = 0; i < nums.size(); i++) {
        // 剪枝：最小的数已经大于 0，不可能再凑出 0
        if (nums[i] > 0) {
            break;
        }

        // 对 a 去重
        if (i > 0 && nums[i] == nums[i - 1]) {
            continue;
        }

        int left = i + 1;
        int right = nums.size() - 1;

        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum > 0) {
                right--;
            } else if (sum < 0) {
                left++;
            } else {
                result.push_back({nums[i], nums[left], nums[right]});
                // 对 b 和 c 去重
                while (left < right && nums[right] == nums[right - 1]) right--;
                while (left < right && nums[left] == nums[left + 1]) left++;
                // 双指针同时向中间收窄
                right--;
                left++;
            }
        }
    }
    return result;
}
```

---

## 四数之和（LeetCode 18）

题目：在一个数组中找到所有不重复的四元组 `[a, b, c, d]`，使得 `a + b + c + d == target`。target 可以是任意整数。

### 思路

三数之和的扩展——多嵌套一层循环。固定前两个数 `nums[k]` 和 `nums[i]`，剩余两个数用双指针在 `[i+1, n-1]` 区间内查找。

### 与三数之和的关键区别

**1. target 不再固定为 0**

这意味着剪枝条件需要更谨慎。

- 三数之和中 `nums[i] > 0` 即可剪枝（因为 target = 0，后面都是正数不可能变小）。
- 四数之和中，target 可以是负数。例如 `nums = [-5, -4, -3, -2], target = -14`，剪掉负数会漏解。

正确的剪枝条件：**在当前数是正数，且 target 是正数，且当前数已经大于 target 时**，才可以剪枝。

```cpp
if (nums[k] > target && nums[k] >= 0 && target >= 0) {
    break;
}
```

对于第二层循环同样需要更谨慎：

```cpp
if (nums[k] + nums[i] > target && nums[k] + nums[i] >= 0) {
    break;
}
```

**2. 溢出问题**

四个数相加可能超过 `int` 范围（例如每个数都是 10⁹ 级别）。需要转为 `long` 再求和：

```cpp
if ((long)nums[k] + nums[i] + nums[left] + nums[right] > target) {
    right--;
}
```

> C++ 中 `(long)nums[k]` 将第一个操作数提升为 `long`，后续加法会自动向 `long` 提升，避免溢出。

**3. 去重**

和三数之和一样：对 `k` 和 `i` 分别用 `nums[x] == nums[x - 1]` 跳过重复值，b 和 c 找到答案后跳过相邻重复值。

### 代码

```cpp
vector<vector<int>> fourSum(vector<int>& nums, int target) {
    vector<vector<int>> result;
    sort(nums.begin(), nums.end());

    for (int k = 0; k < nums.size(); k++) {
        // 一级剪枝
        if (nums[k] > target && nums[k] >= 0 && target >= 0) {
            break;
        }
        // 对 a 去重
        if (k > 0 && nums[k] == nums[k - 1]) {
            continue;
        }

        for (int i = k + 1; i < nums.size(); i++) {
            // 二级剪枝
            if (nums[k] + nums[i] > target && nums[k] + nums[i] >= 0) {
                break;
            }
            // 对 b 去重
            if (i > k + 1 && nums[i] == nums[i - 1]) {
                continue;
            }

            int left = i + 1;
            int right = nums.size() - 1;

            while (left < right) {
                long sum = (long)nums[k] + nums[i] + nums[left] + nums[right];
                if (sum > target) {
                    right--;
                } else if (sum < target) {
                    left++;
                } else {
                    result.push_back({nums[k], nums[i], nums[left], nums[right]});
                    // 对 c 和 d 去重
                    while (left < right && nums[right] == nums[right - 1]) right--;
                    while (left < right && nums[left] == nums[left + 1]) left++;
                    right--;
                    left++;
                }
            }
        }
    }
    return result;
}
```

---

## 总结

| 题目 | 核心方法 | 复杂度 | 易错点 |
|------|----------|--------|--------|
| 有效字母异位词 | 哈希表 / 数组计数 | O(n) | 先判长度 |
| 赎金信 | 哈希表 / 数组计数 | O(n) | 不需要判长度相等 |
| 两个数组的交集 | unordered_set 去重 | O(n+m) | 利用 set 自动去重 |
| 两数之和 | unordered_map 存值→下标 | O(n) | 不能重复用同一元素 |
| 四数相加 II | 分治 + 哈希表统计频率 | O(n²) | 用 find 而非 [] 避免意外插入 |
| 三数之和 | 排序 + 固定一 + 双指针 | O(n²) | 去重用 `nums[i]==nums[i-1]` 而非 `nums[i]==nums[i+1]` |
| 四数之和 | 排序 + 固定二 + 双指针 | O(n³) | 剪枝要考虑负数；四数求和用 long 防溢出 |

哈希表的核心思维模型：**当需要快速判断一个元素是否出现过、或者需要建立映射关系时，第一时间想到哈希表。** 对于 key 范围有限且连续的场景（如小写字母 26 个），数组是更轻量的替代方案。对于要求去重且涉及多元素组合的题目（三数之和、四数之和），排序 + 双指针往往比哈希表更简洁。
