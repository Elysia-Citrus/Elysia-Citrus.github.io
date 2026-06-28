---
title: <题外话>算法练习 Day 11：二叉树回溯与构造——二叉树的所有路径 II、路径总和 II、回溯模式总结、从中序与后序遍历序列构造二叉树
published: 2026-06-28
description: 回溯模式总结（bool / void+result / int 三种返回类型）、显式回溯（push → 递归 → pop）、从 path 的栈帧视角理解回溯、中序+后序构造二叉树（后序末尾为根，中序定位左右子树）
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日焦点

主要做了两件事：一是把 Day 10 中用到的回溯手法做了系统化的归纳，二是用从中序+后序遍历构造二叉树来练习数组区间划分。

回溯方面，Day 10 的“二叉树的所有路径”用了 `string` 值传递的隐式回溯，路径总和用了 `int` 的显式回溯。今天统一换成 `vector<int>& path` 的引用传参 + 显式 `pop_back()`——这是回溯最经典的形式，因为后面排列、组合、子集问题中状态几乎都是 `vector`，不可能全用值传递。关键模板是：

```
path.push_back(val)    → 做选择
recurse(...)           → 递归深入
path.pop_back()        → 撤销选择，回到当前层时 path 恢复原状
```

稍微总结了回溯的三种返回类型：`bool`（问有没有，找到一个即可返回）、`void + result 引用`（要求所有答案）、`int`（求数量，汇总左右子树的满足条件数）。在路径总和.cpp 里画了一下栈帧的示意图——`5 → 4 → 11 → 7` 这个路径在程序栈里是四层 `findTargetPath` 叠在一起，最深处返回 false 后逐层弹出，回到上一层的 `if (root->left)` 里继续尝试另一侧。回溯不是可以被视为是函数调用栈的自然行为。

构造二叉树这道题的核心是理解中序和后序中节点的排列规律。后序的最后一个元素一定是当前子树的根节点；在中序中找到这个根之后，左边就是左子树，右边就是右子树。然后通过 `leftSize = rootIndex - inLeft` 在后序数组中切出对应的左右子树区间。整道题的难点不在算法而在边界计算——一旦 `inLeft`、`inRight`、`postLeft`、`postRight` 四个下标的关系理清了，代码就水到渠成。

---

## 回溯练习 1：二叉树的所有路径（显式回溯版）

Day 10 中这道题用的是 `string path` 值传递——隐式回溯。今天用 `vector<int>& path` 引用传参重写，掌握显式回溯。

两者的差异在于：
- **值传递**（`string path`）：每层递归拿到独立副本，不需要手动恢复，C++ 栈帧替你管理。
- **引用传参**（`vector<int>& path`）：所有递归层共用同一个 `path`，在左子树里 `push_back` 会直接影响右子树，因此必须在离开前 `pop_back`。

```cpp
void dfs(TreeNode* root, vector<int>& path, vector<vector<int>>& result) {
    if (root == nullptr) return;

    path.push_back(root->val);   // 做选择：进入当前节点

    if (isLeaf(root)) {
        result.push_back(path);  // 叶子节点，收集一条完整路径
    }

    dfs(root->left,  path, result);
    dfs(root->right, path, result);

    path.pop_back();             // 撤销选择：离开当前节点，回到父节点
}
```

这里 `pop_back()` 的位置很关键——它必须在左右子树都搜完之后执行。这样从最深层的叶子一路回溯到根节点时，每一层都会弹出自己加入的那个节点值，`path` 逐层恢复到进入该层之前的状态。

### 当前节点写法 vs 子节点写法

回溯练习 3 的笔记里提到了两种写法风格：

- **当前节点写法**（上面用的）：进入 `root` → 处理 `root` → 递归左右 → 撤销 `root`。逻辑直白，推荐。
- **子节点写法**：准备进入左孩子 → 先减去左孩子的值 → 递归 → 如果失败，撤销。这种写法更绕，一般不推荐。

实战中当前节点写法更常用，因为“在进入时处理自己”比“在父节点预判子节点”更符合直觉。

---

## 回溯练习 2：路径总和 II（找所有满足条件的路径）

Day 10 的路径总和是“问有没有”，返回 `bool`。今天这道是“找出所有满足条件的路径”，返回 `vector<vector<int>>`。

思路完全相同，只是收集结果的方式变了：从“找到一个就 `return true`”变成“找到后 `result.push_back` 然后继续搜”。

```cpp
void dfs(TreeNode* root, int target, vector<int>& path, vector<vector<int>>& result) {
    if (root == nullptr) return;

    path.push_back(root->val);   // 做选择
    target -= root->val;         // 更新状态

    if (isLeaf(root) && target == 0) {
        result.push_back(path);  // 收集结果（注意：此时还要继续搜，不能 return）
    }

    dfs(root->left,  target, path, result);
    dfs(root->right, target, path, result);

    path.pop_back();             // 撤销选择
}
```

注意一点：`target` 是值传递的 `int`，所以不需要手动恢复——每个栈帧有自己的 `target` 副本，这和 `path`（引用传参）不同。

---

## 回溯练习 3：回溯模式总结

尝试归纳三种常见返回类型的模板，适用于二叉树的路径类问题，也适用于后面的组合、排列、子集。

### 1. 问有没有 → 返回 `bool`

找到一个就立即返回，不再继续搜索（短路）。

```cpp
bool dfs(TreeNode* root, int target) {
    if (root == nullptr) return false;

    target -= root->val;

    if (isLeaf(root)) {
        return target == 0;
    }

    // 左右子树有一个成功即可
    return dfs(root->left, target) || dfs(root->right, target);
}
```

### 2. 要求全部答案 → `void + result 引用`

需要遍历所有可能性，不能短路。把所有满足条件的路径收集到 `result`。

```cpp
void dfs(TreeNode* root, int target, vector<int>& path, vector<vector<int>>& result) {
    if (root == nullptr) return;

    path.push_back(root->val);
    target -= root->val;

    if (isLeaf(root) && target == 0) {
        result.push_back(path);  // 这里不 return，继续搜
    }

    dfs(root->left,  target, path, result);
    dfs(root->right, target, path, result);

    path.pop_back();
}
```

### 3. 求数量 → 返回 `int`

左右子树的方案数汇总到当前节点，逐层向上累加。

```cpp
int dfs(TreeNode* root, int target) {
    if (root == nullptr) return 0;

    target -= root->val;

    if (isLeaf(root)) {
        return target == 0 ? 1 : 0;
    }

    return dfs(root->left, target) + dfs(root->right, target);
}
```

这三种模式其实就是后序模板在处理不同问题时的变形：从子树拿回结果，在“中”这一步做对应的加工——或者逻辑或（`||`），或者收集结果，或者求和。

---

## 回溯的栈帧视角

更新后的路径总和.cpp 里详细画了递归调用的栈帧结构。以 `5 → 4 → 11 → 7` 这条路径为例，程序运行时的调用栈大概是：

```
findTargetPath(7,  -5)    ← 栈顶：7 是叶子，target=-5≠0，返回 false
findTargetPath(11,  2)    ← 11 收到了左边(7)的 false，接下来尝试右边(2)
findTargetPath(4,  13)    ← 4 的左边(11)还没搜完
findTargetPath(5,  17)    ← 5 的左边(4)还没搜完
main()                    ← 栈底
```

当 `findTargetPath(7, -5)` 返回 false，这一层栈帧弹出，程序回到 `findTargetPath(11, 2)` 的 `if (root->left)` 里面。代码紧接着执行 `target += root->left->val`（回溯），然后去尝试 `root->right`（即节点 2）。

之后 `findTargetPath(2, ...)` — 结果 2 是叶子且 target==0，返回 true。这个 true 一路穿过 `findTargetPath(11) → findTargetPath(4) → findTargetPath(5) → main()`，每个上层看到 `true` 立刻短路返回，不再尝试其他分支。

理解这个栈帧视角后，回溯就不再神秘了——它只是函数调用栈的自然行为：进入子问题前修改状态，从子问题返回后恢复状态，这样兄弟子问题看到的状态就是干净的。

---

## 从中序与后序遍历序列构造二叉树

### 问题

给定中序遍历 `inorder` 和后序遍历 `postorder` 两个数组，构造出原始二叉树。

### 思路

首先要理解两种遍历的排列规律。以后序 `[4, 2, 9, 15, 7, 20, 3]` 和中序 `[4, 9, 2, 3, 15, 20, 7]` 为例：

- **后序**：`[左子树后序] + [右子树后序] + [根]`。最后一个元素 `3` 一定是整棵树的根。
- **中序**：`[左子树中序] + [根] + [右子树中序]`。找到 `3` 的位置，左边 `[4, 9, 2]` 是左子树，右边 `[15, 20, 7]` 是右子树。

关键一步是通过中序里根的位置计算出左子树的节点个数 `leftSize`，然后在后序数组里切出对应的左子树区间和右子树区间。

整道题可以理解为：用四个边界下标 `[inLeft, inRight]` 和 `[postLeft, postRight]` 描述“当前正在构造哪棵子树”，递归地把区间越切越小，直到区间为空。

### 代码

```cpp
TreeNode* build(
    vector<int>& inorder,  int inLeft,  int inRight,
    vector<int>& postorder, int postLeft, int postRight,
    unordered_map<int, int>& indexMap   // 节点值 → 中序数组中的下标
) {
    if (inLeft > inRight) return nullptr;   // 区间为空，空节点

    // 后序的最后一个元素是当前子树的根
    int rootVal = postorder[postRight];
    TreeNode* root = new TreeNode(rootVal);

    // 在中序里找到根的位置（O(1) 查哈希表）
    int rootIndex = indexMap[rootVal];

    // 左子树的节点个数 = 根下标 - 中序左边界
    int leftSize = rootIndex - inLeft;

    // 构造左子树
    // 中序区间：[inLeft, rootIndex - 1]
    // 后序区间：[postLeft, postLeft + leftSize - 1]
    root->left = build(
        inorder,  inLeft,  rootIndex - 1,
        postorder, postLeft, postLeft + leftSize - 1,
        indexMap
    );

    // 构造右子树
    // 中序区间：[rootIndex + 1, inRight]
    // 后序区间：[postLeft + leftSize, postRight - 1]
    root->right = build(
        inorder,  rootIndex + 1, inRight,
        postorder, postLeft + leftSize, postRight - 1,
        indexMap
    );

    return root;
}
```

四个边界下标的含义和计算：

| 边界 | 含义 | 计算方式 |
|------|------|---------|
| 左子树中序左边界 | `inLeft`（不变） | 根左侧就是左子树 |
| 左子树中序右边界 | `rootIndex - 1` | 根的前一个位置 |
| 右子树中序左边界 | `rootIndex + 1` | 根的后一个位置 |
| 右子树中序右边界 | `inRight`（不变） | 根右侧就是右子树 |
| 左子树后序左边界 | `postLeft`（不变） | 后序中左子树紧挨着开头 |
| 左子树后序右边界 | `postLeft + leftSize - 1` | 从左边界往后数 `leftSize` 个 |
| 右子树后序左边界 | `postLeft + leftSize` | 紧接左子树后序之后 |
| 右子树后序右边界 | `postRight - 1` | `postRight` 是根，不能包括 |

画成区间示意（以 `[4, 2, 9, 15, 7, 20, 3]` 为例，`leftSize = 3`）：

```
后序: [ 4,  2,  9,  15,  7,  20,  3 ]
       ↑             ↑             ↑
   postLeft   postLeft+leftSize  postRight
       |___左子树后序___| |_右子树后序_| |根|
```

### 复杂度

- 时间：O(n)。每个节点访问一次，哈希表查找根位置为 O(1)。
- 空间：O(n)。哈希表存储 n 个节点的值-下标映射；递归栈深度最坏 O(n)（树退化为链表）。

### 小结

这道题的难点在于把区间映射关系想清楚。`leftSize` 是连接两个数组的桥梁——通过中序算出左子树有几个节点，就能在后序里找到对应区间。一旦四个边界写对，代码结构就是标准的“后序构造”：先建左右子树递归，最后返回当前根。

这也解释了为什么用中序+后序（或中序+前序）能唯一确定一棵二叉树，但前序+后序不行——没有中序，就无法知道左右子树各有多少节点。
