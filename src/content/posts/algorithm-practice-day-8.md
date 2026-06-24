---
title: <题外话>算法练习 Day 8：二叉树基础、迭代遍历、层序遍历
published: 2026-06-23
description: 二叉树结构定义、递归遍历、迭代遍历（前中后序）、层序遍历
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日说法

主要学习了二叉树的结构定义和几种遍历方式。二叉树由根节点、左子树、右子树构成，天然适合递归。从数组构建二叉树利用下标关系 `2*i+1` 和 `2*i+2`。

遍历分为两类：深度优先（前序、中序、后序）可用递归或栈迭代实现；广度优先（层序）用队列实现。迭代遍历中，前序和后序模板统一（后序只需调换压栈顺序再 reverse），中序需要一路向左走到底再逐个弹出。层序遍历的关键是每层用固定的 `size` 控制循环次数。

## 二叉树结构定义

```cpp
struct TreeNode{
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x): val(x), left(nullptr), right(nullptr) {}
};
```

## 从数组构建二叉树

```cpp
TreeNode* buildTree(vector<int>& nums) {
    if (nums.empty()) return nullptr;

    vector<TreeNode*> nodes;

    for (int x : nums) {
        nodes.push_back(new TreeNode(x));
    }

    for (int i = 0; i < nums.size(); i++) {
        int leftIndex = 2 * i + 1;
        int rightIndex = 2 * i + 2;

        if (leftIndex < nums.size()) {
            nodes[i]->left = nodes[leftIndex];
        }

        if (rightIndex < nums.size()) {
            nodes[i]->right = nodes[rightIndex];
        }
    }

    return nodes[0];
}
```

## 递归遍历

递归的核心在于：

1. 确定递归函数的参数和返回值：确定哪些参数是递归的过程中需要处理的，那么就在递归函数里加上这个参数， 并且还要明确每次递归的返回值是什么进而确定递归函数的返回类型。

2. 确定终止条件： 写完了递归算法, 运行的时候，经常会遇到栈溢出的错误，就是没写终止条件或者终止条件写的不对，操作系统也是用一个栈的结构来保存每一层递归的信息，如果递归没有终止，操作系统的内存栈必然就会溢出。

3. 确定单层递归的逻辑： 确定每一层递归需要处理的信息。在这里也就会重复调用自己来实现递归的过程。

```cpp
//前序遍历, 中左右

void pre_trav(TreeNode* cur, vector<int>& vec) {
    if (cur == nullptr) return;

    cout << cur->val << " ";
    vec.push_back(cur->val);

    pre_trav(cur->left, vec);
    pre_trav(cur->right, vec);
}

int main(){
    vector<int> sample1 = {1, 2, 3, 4, 5, 6, 7};

    TreeNode* root = buildTree(sample1);


    vector<int> ans;
    pre_trav(root, ans);
    return 0;
}
```

## 迭代遍历

递归的实现是：每一次递归调用都会把函数的局部变量、参数值和返回地址等压入调用栈中，然后递归返回的时候，从栈顶弹出上一次递归的各项参数，所以这就是递归为什么可以返回上一层位置的原因。

理论上所有递归的逻辑都可以使用栈实现，但是复杂的递归就没必要了。这里的代码比较简单所以用栈实现。

### 前序遍历：中左右

每次先处理的是中间节点，那么先将根节点放入栈中，然后将右孩子加入栈（因为栈是先进后出），再加入左孩子。

```cpp
vector<int> pre_trav(TreeNode* root) {
    vector<int> vec;

    if (root == nullptr) return vec;

    stack<TreeNode*> st;
    st.push(root);

    while (!st.empty()) {
        TreeNode* cur = st.top();
        st.pop();  // 弹出栈顶

        vec.push_back(cur->val);

        if (cur->right != nullptr) {
            st.push(cur->right);
        }

        if (cur->left != nullptr) {
            st.push(cur->left);
        }
    }

    return vec;
}
```

### 后序遍历：左右中

后序：相比前序，注意的是改变访问顺序，并且reverse一下。

补充：我们是通过抓住节点的指针来访问其左右孩子的。

```cpp
vector<int> pos_trav(TreeNode* root){
    vector<int> vec;

    if(root == nullptr) return vec;

    stack<TreeNode*> st;

    st.push(root);

    while(!st.empty()){
        TreeNode* cur = st.top();
        st.pop();

        vec.push_back(cur->val);

        if(cur->left){
            st.push(cur->left);
        }
        if(cur->right){
            st.push(cur->right);
        }
    }
    reverse(vec.begin(), vec.end());
    return vec;
}
```

### 中序遍历：左中右

中序遍历：相比起其他，更加抽象了一点。循环内需要先把所有的左孩子进栈，然后再从st弹出栈顶，再让指针访问右孩子。

```cpp
vector<int> inorder_trav(TreeNode* root){
    vector<int> vec;
    if(root == nullptr) return vec;

    stack<TreeNode*> st;
    TreeNode* cur = root;

    while(!st.empty() || cur != nullptr){
        while(cur != nullptr){
            st.push(cur); //新一轮开始时压进栈，直到左孩子为空
            cur = cur->left; //先一路往左走
        }
        cur = st.top();
        st.pop();
        vec.push_back(cur->val);
        cur = cur->right;
    }
    return vec;
}

int main() {
    SetConsoleOutputCP(CP_UTF8);
    TreeNode* root = new TreeNode(1);
    root->left = new TreeNode(2);
    root->right = new TreeNode(3);
    root->left->left = new TreeNode(4);
    root->left->right = new TreeNode(5);
    root->right->left = new TreeNode(6);
    root->right->right = new TreeNode(7);
/*
        1
      /   \
     2     3
    / \   / \
   4   5 6   7
*/

    vector<int> ans;

    ans = inorder_trav(root);

    for(int x : ans){
        cout << x << ",";
    }

    return 0;
}
```

## 层序遍历

主要思路：只要队列不空，就取出队伍头节点，访问它，把它的左孩子、右孩子分别加入队列。

使用固定大小size，不使用que.size()，因为que.size是不断变化的。

```cpp
vector<vector<int>> levelOrder(TreeNode* root){
    queue<TreeNode*> que;
    if(root != nullptr) que.push(root);
    vector<vector<int>> result;

    while(!que.empty()){
        int size = que.size();
        vector<int> vec;
        for(int i = 0; i < size; i++){
            TreeNode* cur = que.front();  //获得头节点
            que.pop();
            vec.push_back(cur->val);
            if(cur->left){
                que.push(cur->left);
            }
            if(cur->right){
                que.push(cur->right);
            }
        }
        result.push_back(vec);
    }
    return result;
}

int main() {
    SetConsoleOutputCP(CP_UTF8);
    TreeNode* root = new TreeNode(1);
    root->left = new TreeNode(2);
    root->right = new TreeNode(3);
    root->left->left = new TreeNode(4);
    root->left->right = new TreeNode(5);
    root->right->left = new TreeNode(6);
    root->right->right = new TreeNode(7);
/*
        1
      /   \
     2     3
    / \   / \
   4   5 6   7
*/

    vector<vector<int>> ans;

    ans = levelOrder(root);

    // for(int x : ans){
    //     cout << x << ",";
    // }
    for (auto level : ans) {
        for (int x : level) {
            cout << x << ",";
        }
        cout << endl;
    }

    return 0;
}
```
