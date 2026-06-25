---
title: <题外话>算法练习 Day 9：翻转二叉树、对称二叉树、二叉树的最大深度与最小深度
published: 2026-06-25
description: 翻转二叉树（前序/后序递归）、对称二叉树（递归比较内外侧 + 队列迭代）、二叉树最大深度（后序求高度）、二叉树最小深度（单侧为空需特殊处理）
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日焦点

练习了四道二叉树题目。翻转二叉树的关键是交换指针而非数值，前序或后序遍历最直接，中序会导致同一节点被翻转两次。对称二叉树本质是比较左右子树是否镜像——递归时外侧（left→left 与 right→right）和内侧（left→right 与 right→left）分别比较；迭代时用队列每次弹出两个节点比对。二叉树的最大深度通过后序遍历求根节点高度来实现（左右孩子高度取 max + 1）。二叉树的最小深度与最大深度的核心差异在于：当某个子树为空时，最小深度的路径在另一侧，不能直接用 `min(left, right) + 1`。

## 翻转二叉树

翻转的是指针，不是数值。前序和后序遍历最直接；中序翻转比较难改写，会产生重复翻转。

递归三要素：

1. **返回值与参数**：返回节点指针，传入根节点。
2. **终止条件**：遇到空节点时返回。
3. **单层递归逻辑**（前序为例）：交换左右孩子 `swap(root->left, root->right)`，然后递归翻转左右子树。

```cpp
TreeNode* invertTree(TreeNode* root) {
    if (root == nullptr) return nullptr;

    swap(root->left, root->right);
    invertTree(root->left);
    invertTree(root->right);

    return root;
}
```

层序遍历输出验证：

```cpp
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> result;
    queue<TreeNode*> que;

    if (root != nullptr) que.push(root);

    while (!que.empty()) {
        int size = que.size();
        vector<int> level;

        for (int i = 0; i < size; i++) {
            TreeNode* cur = que.front();
            que.pop();

            level.push_back(cur->val);

            if (cur->left != nullptr) que.push(cur->left);
            if (cur->right != nullptr) que.push(cur->right);
        }

        result.push_back(level);
    }

    return result;
}
```

## 对称二叉树

核心是比较左右子树是否镜像对称。可用递归或迭代实现。

### 递归法

比较内侧和外侧：

- 外侧：左子树的左孩子 vs 右子树的右孩子
- 内侧：左子树的右孩子 vs 右子树的左孩子

```cpp
bool compare(TreeNode* left, TreeNode* right) {
    if (left == NULL && right != NULL) return false;
    else if (left != NULL && right == NULL) return false;
    else if (left == NULL && right == NULL) return true;
    else if (left->val != right->val) return false;

    bool outside = compare(left->left, right->right);
    bool inside = compare(left->right, right->left);
    return outside && inside;
}

bool isSymmetric(TreeNode* root) {
    if (root == nullptr) return true;
    return compare(root->left, root->right);
}
```

### 迭代法（队列）

每次弹出两个节点进行比较，然后将对称位置的子节点成对入队。

```cpp
bool isSymmetric(TreeNode* root) {
    if (root == nullptr) return true;

    queue<TreeNode*> que;
    que.push(root->left);
    que.push(root->right);

    while (!que.empty()) {
        TreeNode* leftNode = que.front();
        que.pop();

        TreeNode* rightNode = que.front();
        que.pop();

        if (leftNode == nullptr && rightNode == nullptr) {
            continue;
        }

        if (leftNode == nullptr || rightNode == nullptr) {
            return false;
        }

        if (leftNode->val != rightNode->val) {
            return false;
        }

        que.push(leftNode->left);
        que.push(rightNode->right);

        que.push(leftNode->right);
        que.push(rightNode->left);
    }

    return true;
}
```

## 二叉树的最大深度

- **深度**：节点到根节点的距离（前序遍历，从上往下）。
- **高度**：节点到叶子节点的距离（后序遍历，从下往上）。

根节点的高度就是二叉树的最大深度，因此用后序遍历求高度即可。

```cpp
int getHeight(TreeNode* root) {
    if (root == nullptr) return 0;

    int leftHeight = getHeight(root->left);
    int rightHeight = getHeight(root->right);

    int maxHeight = max(leftHeight, rightHeight) + 1;
    return maxHeight;
}
```

## 二叉树的最小深度

与最大深度的关键区别：当某个子树为空时，最小深度由另一侧决定，不能直接用 `min(left, right) + 1`。

例如根节点左子树为空、右子树深度为 3，最小深度应是 1 + 3 = 4，而非 1（空子树被 `min` 误选为 0）。

```cpp
int getDepth(TreeNode* root) {
    if (root == nullptr) return 0;

    int leftD = getDepth(root->left);
    int rightD = getDepth(root->right);

    // 左子树为空，走右子树
    if (root->left == NULL && root->right != NULL) {
        return 1 + rightD;
    }
    // 右子树为空，走左子树
    if (root->left != NULL && root->right == NULL) {
        return 1 + leftD;
    }

    return 1 + min(leftD, rightD);
}
```
