---
title: 算法练习 Day 10：平衡二叉树、二叉树的所有路径、左叶子之和、完全二叉树的节点个数、找树左下角的值、路径总和
published: 2026-06-26
description: 平衡二叉树（后序返回 -1 剪枝）、二叉树的所有路径（前序隐式回溯）、左叶子之和（父节点判定）、完全二叉树的节点个数（满二叉树公式 2^h-1 加速）、找树左下角的值（层序取最后层首元素）、路径总和（前序显式回溯）
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日复盘

今天做的六道题围绕一个共同的核心问题：**递归时，节点信息应该从父节点往子节点传，还是从子节点往父节点归？** 这决定了你选前序还是后序。

平衡二叉树是 Day 9 “二叉树的最大深度” 的延伸。在求高度的同时加入平衡判断，一旦 `abs(左高 - 右高) > 1` 就返回 -1 作为“已不平衡”的信号，上层看到 -1 直接继续向上返回，不再做多余计算。这个 -1 标记的技巧很实用——它把“判断”和“计算高度”合并到了一个函数里，不用额外写一个 `isBalanced` 去反复遍历。

二叉树的所有路径让我第一次清晰地感受到“隐式回溯”。`path` 参数是值传递（`string` 而非 `string&`），所以每层递归都持有自己的一份独立副本。往 `path` 后面拼接 `"->" + val` 只影响当前这一层；函数返回后，父节点的 `path` 还是原来的样子。没有显式的 `path.pop()` 或 `target +=`，回溯却已经完成了——因为 C++ 的栈帧替你管理了这份拷贝。递归的“归”本身就是回溯，这句话在这里很具象。

左叶子之和教会我一个容易踩的坑：叶子节点自己的视角无法判断自己是“左”还是“右”，必须由父节点来判定。所以 `isLeftLeaf(node)` 检查的是 `node->left` 是不是叶子，而不是 `node` 本身。整体用后序：先收集左右子树中的左叶子之和，再看当前节点的左孩子是不是左叶子，三层结果相加。

完全二叉树的节点个数有两种做法。通用后序计数谁都会写，但完全二叉树的特性可以让时间复杂度从 O(n) 降到 O(log²n)：如果某个子树一路向左的深度等于一路向右的深度，那它就是一棵满二叉树，节点数直接套公式 `2^h - 1`，不用继续往下拆。我第一次写的时候犯了个错——在 `nullptr` 处 +1 计数，结果发现答案总是比正确值多 1。后来才意识到二叉树的性质：**空指针数量 = 节点数量 + 1**。标准做法是在每个真实节点处 +1，空节点返回 0 不贡献任何值。

找树左下角的值是今天最简单的一道，层序遍历拿到最后一层的 vector，取第一个元素就行。层序遍历天然按层从左到右，所以最后一层的第一个就是树左下角。

路径总和让我第一次手写了显式回溯：进入子节点前 `target -= child->val`，返回后 `target += child->val`。这里如果不恢复 target，回到父节点走另一条分支时 target 的值就不对了。这和“二叉树的所有路径”形成了对照——一个是隐式回溯（值传递），一个是显式回溯（手动恢复状态）。两种方式都能正确回溯，选择哪种取决于参数类型和性能考量。

---

## 前序 vs 后序：两种思维模式

经过 Day 8、Day 9 和今天的练习，可以明确地把二叉树递归问题分成两类：

| 遍历顺序 | 思维方向 | 典型场景 | 今天涉及 |
|---------|---------|---------|---------|
| 前序（中左右） | 我先处理当前节点，再把某种状态传递给左右孩子 | 翻转二叉树、记录路径、传递当前深度/剩余target | 二叉树的所有路径、路径总和 |
| 后序（左右中） | 我先拿到左右子树的结果，再加工当前节点返回给父节点 | 求高度/深度、判断平衡、汇总子树信息 | 平衡二叉树、左叶子之和、完全二叉树节点个数（通用版） |

后序遍历可以抽象出一个非常通用的模板，今天的好几道题都是它的变形：

```cpp
返回值类型 dfs(TreeNode* root) {
    if (root == nullptr) {
        return 空节点对应的信息;  // 比如 0 或 nullptr
    }

    返回值类型 leftInfo  = dfs(root->left);
    返回值类型 rightInfo = dfs(root->right);

    返回值类型 curInfo;
    // 根据 leftInfo、rightInfo、root->val 三者加工出当前节点的信息
    curInfo = ...;

    return curInfo;
}
```

- 求最大深度：`return 1 + max(left, right)`
- 判断平衡：`if (abs(left - right) > 1) return -1; else return 1 + max(left, right)`
- 左叶子之和：`return leftSum + rightSum + (当前节点的左孩子是左叶子 ? 左孩子.val : 0)`
- 完全二叉树节点个数：`return leftCount + rightCount + 1`

你会发现这些题目的骨架完全一样，区别只在于“加工 `curInfo`”这一步的逻辑。

---

## 平衡二叉树

### 问题

判断一棵二叉树是否是高度平衡的——任意节点的左右子树高度差不超过 1。

### 思路

这道题是 Day 9 “二叉树的最大深度” 的自然延伸。最大深度用后序求根节点高度，平衡二叉树在求高度的过程中顺手判断是否平衡。

关键设计：让 `getHeight` 承担双重职责——正常情况下返回高度，发现不平衡时返回 -1。上层调用者一旦收到 -1，就知道下面已经不平衡了，自己也直接返回 -1，不再往下计算。这是一种**剪枝**：最早发现不平衡的位置就会中断，避免整棵树都被遍历。

为什么用后序？因为你需要先知道左右子树分别是否平衡、高度分别是多少，才能判断当前节点是否平衡。这是典型的“从下往上收集信息”的场景。

### 代码

```cpp
// 返回 -1 表示已经不平衡，否则返回当前节点的高度
int getHeight(TreeNode* root) {
    if (root == nullptr) return 0;

    // 左
    int leftHeight = getHeight(root->left);
    if (leftHeight == -1) return -1;  // 左子树已经不平衡，直接向上传递

    // 右
    int rightHeight = getHeight(root->right);
    if (rightHeight == -1) return -1;  // 右子树已经不平衡，直接向上传递

    // 中：判断当前节点是否平衡
    if (abs(leftHeight - rightHeight) > 1) return -1;

    // 平衡，返回正常高度
    return 1 + max(leftHeight, rightHeight);
}

bool isBalanced(TreeNode* root) {
    return getHeight(root) != -1;
}
```

### 小结

把“判断”和“计算”合并到一个返回值里（用 -1 作哨兵）是常见的编程技巧。如果不这样做，就需要写两个函数——一个算高度、一个判平衡——对每个节点都要调两次，代码冗余且多一次遍历。

另外注意 `abs(leftHeight - rightHeight)` 用的是 `<cmath>` 或 `<cstdlib>` 中的 `abs`。如果两数之差超过了 `int` 的表达范围可能会溢出，但在树高的场景下不会，因为高度差本身不会超过树高。

---

## 二叉树的所有路径

### 问题

给定一棵二叉树，返回所有从根节点到叶子节点的路径（格式：`"1->2->5"`）。

### 思路

这道题是前序遍历的典型应用：从根往下走，每经过一个节点就把它拼接到路径后面，走到叶子时这条路径就完整了。

它让我第一次清晰地体会到“隐式回溯”。来看函数签名：

```cpp
void treePath(TreeNode* root, string path, vector<string>& result)
```

`path` 是 `string` 类型，按值传递。这意味着每一次递归调用都会复制一份 `path`。当前层往 `path` 后面拼接 `"->" + val`，只影响当前层及以下递归的副本，不影响父节点的 `path`。当函数返回（“归”）时，当前层的 `path` 副本被销毁，父节点的 `path` 还是进入子节点之前的样子——回溯就自动完成了。

如果没有这个值传递的拷贝机制，就需要像路径总和那道题一样手动做显式回溯（先 `target -= val`，返回后再 `target += val`）。

### 代码

```cpp
void treePath(TreeNode* root, string path, vector<string>& result) {
    if (root == nullptr) return;

    // 把当前节点拼到路径后面
    if (path.empty()) {
        path += to_string(root->val);     // 根节点，前面不加箭头
    } else {
        path += "->" + to_string(root->val);
    }

    // 如果是叶子节点，说明一条完整路径已经形成
    if (root->left == nullptr && root->right == nullptr) {
        result.push_back(path);
        return;
    }

    // 继续往左右子树走
    // 注意：这里的 path 是值传递，所以下面两次调用各自拿到的是独立的副本
    treePath(root->left,  path, result);
    treePath(root->right, path, result);
}

vector<string> binaryTreePaths(TreeNode* root) {
    vector<string> result;
    string path = "";           // 初始为空字符串
    treePath(root, path, result);
    return result;
}
```

### 回溯的本质

很多人初学回溯时会纠结“我到底在哪里 pop 了”。这道题给出了另一个视角：

- **显式回溯**：修改共享状态 → 递归 → 恢复共享状态（如路径总和中的 `target -= val` 后 `target += val`）
- **隐式回溯**：每次递归持有状态的独立副本，依靠函数调用栈的销毁自动恢复（如本题的 `path` 值传递）

两种方式等价，选择哪一种取决于：
- 状态是否容易拷贝（`string` 拷贝成本可接受，但 `vector<int>` 拷贝可能较贵）
- 是否需要保留中间状态用于其他逻辑

回溯和递归是相辅相成的。递归的“递”是深入，“归”是回溯。在后面的回溯专题中还会遇到更复杂的场景（组合、排列、子集），但二叉树的所有路径是感受回溯最自然的起点。

---

## 左叶子之和

### 问题

计算二叉树中所有左叶子节点的值之和。左叶子定义为：它是某个节点的左孩子，且自身没有左右孩子。

### 思路

这道题最容易踩的坑是**判断逻辑**。你不能站在叶子节点自己的视角去判断自己是不是“左叶子”——叶子节点不知道自己到底是父节点的左孩子还是右孩子。能做出这个判断的只有父节点。

所以 `isLeftLeaf` 的参数是父节点 `node`，检查的是 `node->left` 是不是叶子：

```cpp
bool isLeftLeaf(TreeNode* node) {
    // 检查 node 的左孩子是否存在，且左孩子是不是叶子
    if (node->left != nullptr
        && node->left->left == nullptr
        && node->left->right == nullptr) {
        return true;
    }
    return false;
}
```

整体结构是后序：
1. 递归去左子树找左叶子之和
2. 递归去右子树找左叶子之和
3. 在“中”这一步，判断当前节点的左孩子是不是左叶子，如果是就加上它的值
4. 三层结果相加返回

为什么不直接只搜左子树？因为右子树里也可能包含左叶子——比如右孩子的左孩子是叶子，它同样是左叶子。

### 代码

```cpp
int sumOfLeftLeaves(TreeNode* root) {
    if (root == nullptr) return 0;

    // 左：左子树中的左叶子之和
    int leftSum = sumOfLeftLeaves(root->left);
    // 右：右子树中的左叶子之和
    int rightSum = sumOfLeftLeaves(root->right);

    // 中：检查当前节点的左孩子是否恰好是左叶子
    int midValue = 0;
    if (isLeftLeaf(root)) {
        midValue = root->left->val;
    }

    return leftSum + rightSum + midValue;
}
```

### 小结

这道题可以看作是后序模板的一次“加工层”练习。`curInfo` 的加工逻辑是：`leftSum + rightSum + (当前节点的左孩子是左叶子 ? 左孩子.val : 0)`。

判断左叶子依赖父节点——这个思路在后面的二叉树题目中还会反复出现，比如删除节点、修剪二叉树等操作往往也需要从父节点出发。

---

## 完全二叉树的节点个数

### 问题

给一棵完全二叉树，计算节点个数。要求利用完全二叉树的性质给出优于 O(n) 的解法。

### 通用解法（O(n)，任意二叉树）

后序模板的直接应用——每个节点贡献 1，加左右子树的节点数：

```cpp
int nodeCounter(TreeNode* root) {
    if (root == nullptr) return 0;
    int leftCount  = nodeCounter(root->left);
    int rightCount = nodeCounter(root->right);
    return leftCount + rightCount + 1;
}
```

我第一次写的时候犯了一个错：在空节点处 `return 1`，想着每个 `nullptr` 也算一个空指针。结果发现答案永远比正确值大 1。这其实恰好印证了二叉树的一个性质：**空指针数量 = 节点数量 + 1**。标准做法是空节点返回 0（不贡献），每个真实节点贡献 1。

### 利用完全二叉树性质加速（O(log²n)）

完全二叉树的一个重要特征：很多子树都是满二叉树。而满二叉树的节点数可以直接用公式计算，不需要遍历每一个节点：

```
满二叉树节点数 = 2^h - 1   （h 为高度，从 1 开始计）
```

怎么判断一棵子树是满二叉树？一路向左走到底得到 `leftHeight`，一路向右走到底得到 `rightHeight`。如果两者相等，说明这棵子树每一层都被填满了（在完全二叉树的前提下），它就是满二叉树。

```cpp
int countNodes(TreeNode* root) {
    if (root == nullptr) return 0;

    // 计算一路向左的深度
    TreeNode* left = root;
    int leftHeight = 0;
    while (left != nullptr) {
        leftHeight++;
        left = left->left;
    }

    // 计算一路向右的深度
    TreeNode* right = root;
    int rightHeight = 0;
    while (right != nullptr) {
        rightHeight++;
        right = right->right;
    }

    // 左右深度相等 → 满二叉树 → 直接套公式
    if (leftHeight == rightHeight) {
        return (1 << leftHeight) - 1;   // 2^h - 1，位运算替代 pow
    }

    // 不是满二叉树，继续递归拆分子树
    return countNodes(root->left) + countNodes(root->right) + 1;
}
```

这里用 `1 << leftHeight` 替代 `pow(2, leftHeight)`，因为整数位移比浮点 `pow` 更快且没有精度问题。

### 复杂度为什么是 O(log²n)

完全二叉树的高度是 O(log n)。每一层最多只会有一棵子树不是满二叉树（非满的那棵需要继续递归），其他全是满二叉树直接 O(1) 出结果。所以递归深度为 O(log n)，每层统计左右深度各 O(log n)，总复杂度 O(log n × log n) = O(log²n)。

### 小结

这道题的启发是：**利用数据结构的特殊性质可以把通用算法的复杂度降下来**。通用后序是 O(n)，利用“完全二叉树 → 很多子树是满二叉树 → 满二叉树可以直接算”这一个链条，就降到了 O(log²n)。类似的思路在后面的二叉搜索树题目中也会遇到（利用 BST 的有序性来剪枝）。

---

## 找树左下角的值

### 问题

找到二叉树最底层最左边的节点的值。

### 思路

层序遍历天然从上到下、从左到右。每一层的结果是一个 `vector<int>`，最后一层的第一个元素就是答案。

这道题花了不到五分钟——层序遍历的模板已经在 Day 8 练熟了，这里只是多了一步“取最后一层的第一个”。这也说明基础遍历模板熟练之后，很多题只是对遍历结果做一步简单的后处理。

### 代码

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
            if (cur->left)  que.push(cur->left);
            if (cur->right) que.push(cur->right);
        }
        result.push_back(vec);
    }

    // 取最后一层（result.back()）的第一个元素（.front()）
    return result.back().front();
}
```

如果不想存所有层的结果（省空间），也可以在层序遍历时只保留当前层，最后一轮循环结束时 `vec[0]` 就是答案。但存下来代码更直观，对于一般数据量也完全够用。

---

## 路径总和

### 问题

判断二叉树中是否存在一条从根到叶子的路径，路径上节点值之和等于给定的 `targetSum`。

### 思路

这道题和“二叉树的所有路径”有相似之处——都需要从根走到叶子，沿途携带状态。区别在于：
- 二叉树的所有路径：携带的是路径字符串，隐式回溯（值传递）
- 路径总和：携带的是剩余 target，显式回溯（手动恢复）

为什么这里用显式回溯？因为 `target` 是 `int`，如果用值传递自然也是隐式回溯。但这里我想练一下显式回溯的手感——在后面的回溯专题中，很多场景必须手动恢复状态（比如修改 `vector`、修改 `used` 数组），这道题是一个温和的入门。

显式回溯的模式：

```
修改状态 → 递归 → 恢复状态
target -= child->val;
dfs(child, target);
target += child->val;   // 恢复，让兄弟节点拿到正确的 target
```

### 代码

```cpp
bool isLeaf(TreeNode* root) {
    return root->left == nullptr && root->right == nullptr;
}

bool findTargetPath(TreeNode* root, int target) {
    // 到达叶子节点，检查 target 是否刚好减到 0
    if (isLeaf(root) && target == 0) {
        return true;
    }
    if (isLeaf(root)) {
        return false;
    }

    // 尝试走左子树
    if (root->left) {
        target -= root->left->val;
        if (findTargetPath(root->left, target)) return true;
        target += root->left->val;  // 回溯：恢复 target
    }

    // 尝试走右子树
    if (root->right) {
        target -= root->right->val;
        if (findTargetPath(root->right, target)) return true;
        target += root->right->val;  // 回溯：恢复 target
    }

    return false;
}

bool hasPathSum(TreeNode* root, int targetSum) {
    if (root == nullptr) return false;
    // 先把根节点的值减掉，然后进入递归
    return findTargetPath(root, targetSum - root->val);
}
```

### 回溯的另一种写法

也可以写成不修改 `target` 的版本（值传递，隐式回溯）：

```cpp
bool hasPathSum(TreeNode* root, int targetSum) {
    if (root == nullptr) return false;
    if (isLeaf(root)) return targetSum == root->val;
    return hasPathSum(root->left,  targetSum - root->val)
        || hasPathSum(root->right, targetSum - root->val);
}
```

这个版本更简洁——每次递归传 `targetSum - root->val`，不修改原变量，自然不需要恢复。两种写法没有对错，取决于你更习惯哪种思维模型。显式回溯在状态复杂（比如要修改一个 `vector` 或 `unordered_set`）时几乎是唯一选择，所以从简单题开始建立手感是值得的。

### 小结

路径总和是今天和“二叉树的所有路径”呼应的题目。两者本质上都是“从根到叶子携带状态”，只是状态类型不同（string vs int）以及回溯方式不同（隐式 vs 显式）。理解这两道题的双生关系，回溯就不是一个神秘的概念了。
