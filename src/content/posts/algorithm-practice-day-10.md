---
title: <题外话>算法练习 Day 10：平衡二叉树、二叉树的所有路径、左叶子之和、完全二叉树的节点个数、找树左下角的值、路径总和
published: 2026-06-26
description: 平衡二叉树（后序返回 -1 剪枝）、二叉树的所有路径（前序隐式回溯）、左叶子之和（父节点判定）、完全二叉树的节点个数（满二叉树公式 2^h-1 加速）、找树左下角的值（层序取最后层首元素）、路径总和（前序显式回溯）
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日糊点

主要练习了六道二叉树题目。平衡二叉树用后序遍历从下往上返回高度，一旦发现不平衡立刻返回 -1 剪枝。

二叉树的所有路径用前序遍历，每层递归持有独立的 `path` 副本（值传递），天然实现了隐式回溯。左叶子之和的关键在于左叶子必须通过父节点来判定（`node->left` 是叶子才算），用后序汇总左右子树结果。

完全二叉树的节点个数除了通用后序计数外，可利用满二叉树性质 `节点数 = 2^h - 1` 加速——当某子树向左和向右的深度相同时，直接套公式而不必递归到底。

找树左下角的值用层序遍历取最后一层的第一个元素最简洁，当然也可以用递归实现，我嫌麻烦就跳过了，以后补以下。

路径总和用前序 + 显式回溯（`target -= val` 再 `target += val`），到达叶子时检查 target 是否恰好减到 0。

前序与后序的思维对照：

| 遍历顺序 | 思维方向 | 常见题目 |
|---------|---------|---------|
| 前序 | 先处理当前节点，再把状态传给子节点 | 翻转二叉树、二叉树的所有路径、路径总和 |
| 后序 | 先拿到左右子树结果，再加工当前节点 | 最大深度、最小深度、平衡二叉树、左叶子之和、完全二叉树节点个数 |

后序的通用模板：

```cpp
返回值类型 dfs(TreeNode* root) {
    if (root == nullptr) return 空节点对应的信息;

    返回值类型 leftInfo = dfs(root->left);
    返回值类型 rightInfo = dfs(root->right);

    返回值类型 curInfo;
    curInfo = 根据 leftInfo、rightInfo、root->val 加工;

    return curInfo;
}
```

## 平衡二叉树

后序遍历求高度，同时判断是否平衡。不平衡时返回 -1 作为标记，上层发现 -1 直接继续返回 -1，起到剪枝效果。

```cpp
int getHeight(TreeNode* root) {
    if (root == nullptr) return 0;

    int leftHeight = getHeight(root->left);
    if (leftHeight == -1) return -1;

    int rightHeight = getHeight(root->right);
    if (rightHeight == -1) return -1;

    if (abs(leftHeight - rightHeight) > 1) return -1;

    return 1 + max(leftHeight, rightHeight);
}

bool isBalanced(TreeNode* root) {
    return getHeight(root) != -1;
}
```

## 二叉树的所有路径

前序遍历，每层递归持有独立的 `path` 字符串（值传递），到达叶子时将完整路径存入结果。因为每层有自己独立的 `path` 副本，函数返回后父节点的 `path` 不受影响——这就是隐式回溯。

```cpp
void treePath(TreeNode* root, string path, vector<string>& result) {
    if (root == nullptr) return;

    if (path.empty()) {
        path += to_string(root->val);
    } else {
        path += "->" + to_string(root->val);
    }

    if (root->left == nullptr && root->right == nullptr) {
        result.push_back(path);
        return;
    }

    treePath(root->left, path, result);
    treePath(root->right, path, result);
}

vector<string> binaryTreePaths(TreeNode* root) {
    vector<string> result;
    treePath(root, "", result);
    return result;
}
```

回溯和递归是相辅相成的——递归的“归”本身就是回溯。

## 左叶子之和

左叶子的判定必须通过父节点：`node->left` 存在且 `node->left` 的左右孩子均为空。后序遍历汇总左右子树中的左叶子值。

```cpp
bool isLeftLeaf(TreeNode* node) {
    if (node->left != nullptr && node->left->left == nullptr && node->left->right == nullptr) {
        return true;
    }
    return false;
}

int sumOfLeftLeaves(TreeNode* root) {
    if (root == nullptr) return 0;

    int leftSum = sumOfLeftLeaves(root->left);
    int rightSum = sumOfLeftLeaves(root->right);

    int midValue = 0;
    if (isLeftLeaf(root)) {
        midValue = root->left->val;
    }

    return leftSum + rightSum + midValue;
}
```

## 完全二叉树的节点个数

### 通用解法

后序遍历，对任意二叉树都适用：

```cpp
int nodeCounter(TreeNode* root) {
    if (root == nullptr) return 0;
    return nodeCounter(root->left) + nodeCounter(root->right) + 1;
}
```

### 利用完全二叉树性质加速

对于完全二叉树，很多子树是满二叉树。判断方法：一路向左的深度等于一路向右的深度时，该子树是满二叉树，节点数直接用公式 `2^h - 1` 计算，无需递归到底。

```cpp
int countNodes(TreeNode* root) {
    if (root == nullptr) return 0;

    TreeNode* left = root;
    TreeNode* right = root;
    int leftHeight = 0;
    int rightHeight = 0;

    while (left != nullptr) {
        leftHeight++;
        left = left->left;
    }
    while (right != nullptr) {
        rightHeight++;
        right = right->right;
    }

    if (leftHeight == rightHeight) {
        return (1 << leftHeight) - 1;
    }

    return countNodes(root->left) + countNodes(root->right) + 1;
}
```

一个小细节：空指针数 = 节点数 + 1。如果写成在每个 `nullptr` 处 +1，结果会比真实节点数多 1。

## 找树左下角的值

层序遍历，取最后一层的第一个元素即可。

```cpp
int findBottomLeftValue(TreeNode* root) {
    queue<TreeNode*> que;
    if (root != nullptr) que.push(root);
    vector<vector<int>> result;

    while (!que.empty()) {
        int size = que.size();
        vector<int> vec;
        for (int i = 0; i < size; i++) {
            TreeNode* cur = que.front();
            que.pop();
            vec.push_back(cur->val);
            if (cur->left) que.push(cur->left);
            if (cur->right) que.push(cur->right);
        }
        result.push_back(vec);
    }

    return result.back().front();
}
```

## 路径总和

前序遍历，每次进入子节点前减去子节点的值，到达叶子时检查是否刚好减到 0。从子节点返回后把值加回来——显式回溯。

```cpp
bool findTargetPath(TreeNode* root, int target) {
    if (isLeaf(root) && target == 0) {
        return true;
    }
    if (isLeaf(root)) {
        return false;
    }

    if (root->left) {
        target -= root->left->val;
        if (findTargetPath(root->left, target)) return true;
        target += root->left->val;  // 回溯
    }

    if (root->right) {
        target -= root->right->val;
        if (findTargetPath(root->right, target)) return true;
        target += root->right->val;  // 回溯
    }

    return false;
}

bool hasPathSum(TreeNode* root, int targetSum) {
    if (root == nullptr) return false;
    return findTargetPath(root, targetSum - root->val);
}
```
