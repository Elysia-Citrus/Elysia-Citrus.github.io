---
title: <题外话>算法练习 Day 7：KMP、栈与队列
published: 2026-06-22
description: KMP字符串匹配、栈与队列基础、用栈实现队列、用队列实现栈、有效的括号、删除相邻重复项
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日导播

主要学习了 KMP 字符串匹配算法，以及栈和队列的基本操作与典型应用。

KMP 的核心思想是在匹配失败时，利用已经匹配成功的部分信息来避免从头开始。通过构建 next 数组记录模式串中每个位置的最长相等前后缀长度，当文本串与模式串在某处失配时，模式串不需要回到起点，而是跳到 next 记录的位置继续匹配，将时间复杂度从暴力法的 O(m×n) 降到 O(m+n)。

栈的特点是后进先出（LIFO），适合处理括号匹配、相邻重复项删除等“最近匹配/最近消除”类问题。队列的特点是先进先出（FIFO）。栈与队列的相互实现是理解两者特性的经典题目——用两个栈实现队列时，一个负责入队，一个负责出队；用单个队列实现栈时，每次 push 后将前面的元素全部旋转到新元素之后，保证队头始终是“栈顶”。

## KMP 算法

KMP 算法用于在文本串中查找模式串首次出现的位置。暴力法在失配时会让模式串回退到起点、文本串回退到下一个位置重新比较，O(m×n) 的复杂度在长字符串场景下不可接受。

KMP 的改进思路：当匹配到模式串的第 j 个字符发现不匹配时，模式串中 `[0, j-1]` 这一段是已经匹配成功的。如果这一段里存在“相等的前缀和后缀”，那么前缀部分一定和后缀部分相同，而文本串中对应的后缀部分也已经匹配过了——这意味着前缀部分天然就能和文本串对应位置匹配上，不需要重新比较。因此模式串可以直接跳到前缀的下一个位置继续匹配，跳过的步数就是最长相等前后缀的长度。

### next 数组

next 数组记录模式串每个位置的最长相等前后缀长度。构建过程使用双指针：`j` 指向前缀末尾（也是当前已匹配的最长相等前后缀长度），`i` 指向后缀末尾并遍历模式串。

- `next[0] = 0`（单个字符没有前后缀）
- 当 `s[i] == s[j]` 时，`j++` 并记录 `next[i] = j`
- 当 `s[i] != s[j]` 且 `j > 0` 时，将 `j` 回退到 `next[j-1]`（利用已有的 next 信息，避免 `j` 直接归零）
- 当 `s[i] != s[j]` 且 `j == 0` 时，`next[i] = 0`

### 匹配过程

双指针 `i`（遍历文本串）和 `j`（指向模式串当前要匹配的位置）。匹配逻辑与构建 next 数组高度对称——当字符相等时 `j++`；当失配且 `j > 0` 时，`j = next[j-1]`；当 `j` 等于模式串长度时，说明找到了完整匹配，返回起始下标。

代码：

```cpp
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>
using namespace std;

void getNext(vector<int>& next, const string& s) {
    next[0] = 0;
    int j = 0;  // j 表示当前已经匹配上的最长相等前后缀长度

    for (int i = 1; i < s.size(); i++) {
        while (j > 0 && s[i] != s[j]) {
            j = next[j - 1];
        }

        if (s[i] == s[j]) {
            j++;
        }

        next[i] = j;
    }
}

int strStr(string haystack, string needle) {
    if (needle.size() == 0) {
        return 0;
    }

    vector<int> next(needle.size());
    getNext(next, needle);

    int j = 0;  // j 指向模式串 needle 当前要匹配的位置

    for (int i = 0; i < haystack.size(); i++) {
        // 当前字符不匹配且 j > 0 时，利用 next 回退 j
        while (j > 0 && haystack[i] != needle[j]) {
            j = next[j - 1];
        }

        if (haystack[i] == needle[j]) {
            j++;
        }

        if (j == needle.size()) {
            return i - needle.size() + 1;
        }
    }

    return -1;
}

int main() {
    string haystack = "aabaabaaf";
    string needle = "aabaaf";

    cout << strStr(haystack, needle) << endl;

    return 0;
}
```

## 栈与队列基础

### 栈（stack）

栈是后进先出（LIFO）的数据结构，只能在一端（栈顶）进行插入和删除。C++ 标准库中 `std::stack` 的常用操作：

| 操作 | 说明 |
|------|------|
| `push(x)` | 入栈 |
| `pop()` | 出栈（不返回值） |
| `top()` | 返回栈顶元素 |
| `empty()` | 判断是否为空 |
| `size()` | 返回元素个数 |

栈不支持迭代器，遍历需要复制一份再逐个弹出：

```cpp
#include <stack>
#include <iostream>
using namespace std;

int main() {
    stack<int> st;

    st.push(10);
    st.push(20);
    st.push(30);

    if (!st.empty()) {
        cout << "top = " << st.top() << endl;  // 30
    }

    st.pop();

    cout << "size = " << st.size() << endl;   // 2

    // 遍历栈
    stack<int> temp = st;
    while (!temp.empty()) {
        cout << temp.top() << " ";
        temp.pop();
    }
    cout << endl;

    return 0;
}
```

### 队列（queue）

队列是先进先出（FIFO）的数据结构，在一端（队尾）插入、另一端（队头）删除。C++ 标准库中 `std::queue` 的常用操作：

| 操作 | 说明 |
|------|------|
| `push(x)` | 入队 |
| `pop()` | 出队（不返回值） |
| `front()` | 返回队头元素 |
| `back()` | 返回队尾元素 |
| `empty()` | 判断是否为空 |
| `size()` | 返回元素个数 |

## 用栈实现队列

用两个栈模拟队列的先进先出。核心思路：栈 A（inStack）负责入队，栈 B（outStack）负责出队。当需要出队且 outStack 为空时，将 inStack 中所有元素逐一弹出并压入 outStack——这样最早入栈的元素就会出现在 outStack 的栈顶，恰好对应队列的队头。

没有必要在每次 push/pop 时都倒腾，而是延迟到真正需要出队（outStack 为空）时才把 inStack 导入 outStack，保证每个元素最多被移动一次。

代码：

```cpp
#include <bits/stdc++.h>
using namespace std;

class MyQueue {
private:
    stack<int> inStack;   // 负责入队
    stack<int> outStack;  // 负责出队

    void moveInToOut() {
        if (outStack.empty()) {
            while (!inStack.empty()) {
                outStack.push(inStack.top());
                inStack.pop();
            }
        }
    }

public:
    MyQueue() {}

    void push(int x) {
        inStack.push(x);
    }

    int pop() {
        moveInToOut();
        int ans = outStack.top();
        outStack.pop();
        return ans;
    }

    int peek() {
        moveInToOut();
        return outStack.top();
    }

    bool empty() {
        return inStack.empty() && outStack.empty();
    }
};

int main() {
    MyQueue q;

    q.push(1);
    q.push(2);

    cout << q.peek() << endl;   // 1
    cout << q.pop() << endl;    // 1
    cout << q.empty() << endl;  // 0

    q.push(3);
    q.push(4);

    cout << q.pop() << endl;    // 2
    cout << q.pop() << endl;    // 3
    cout << q.peek() << endl;   // 4
    cout << q.pop() << endl;    // 4
    cout << q.empty() << endl;  // 1

    return 0;
}
```

## 用队列实现栈

用单个队列模拟栈的后进先出。核心思路：每次 push 一个新元素 x 后，将 x 前面的 n-1 个元素依次从队头弹出再重新入队。这样刚 push 的元素始终在队头位置，队列的 `front()` 就等价于栈的 `top()`。

代码：

```cpp
#include <bits/stdc++.h>
using namespace std;

class MyStack {
private:
    queue<int> q;

public:
    MyStack() {}

    void push(int x) {
        q.push(x);
        int n = q.size();
        // 把 x 前面的 n-1 个元素移动到队尾
        for (int i = 0; i < n - 1; i++) {
            q.push(q.front());
            q.pop();
        }
    }

    int pop() {
        int ans = q.front();
        q.pop();
        return ans;
    }

    int top() {
        return q.front();
    }

    bool empty() {
        return q.empty();
    }
};

int main() {
    MyStack st;

    st.push(1);
    st.push(2);

    cout << st.top() << endl;   // 2
    cout << st.pop() << endl;   // 2
    cout << st.empty() << endl; // 0

    st.push(3);
    st.push(4);

    cout << st.top() << endl;   // 4
    cout << st.pop() << endl;   // 4
    cout << st.pop() << endl;   // 3
    cout << st.pop() << endl;   // 1
    cout << st.empty() << endl; // 1

    return 0;
}
```

## 有效的括号

给定一个只包含 `()`、`{}`、`[]` 的字符串，判断括号是否有效配对且嵌套正确。

利用栈的 LIFO 特性：遇到左括号时，将对应的右括号压入栈；遇到右括号时，检查栈顶是否与之匹配。如果匹配则弹出，不匹配或者栈为空则说明括号无效。最后检查栈是否为空——不为空说明有左括号没有配对。

一个小技巧：遇到左括号时直接 push 对应的右括号，这样在匹配时只需判断 `st.top() == s[i]`，不需要再做三种括号的对照逻辑。

还有一个前置优化：如果字符串长度为奇数，不可能完全配对，直接返回 false。

代码：

```cpp
#include <bits/stdc++.h>
using namespace std;

bool isValid(string s) {
    if (s.size() % 2 != 0) return false;

    stack<char> st;

    for (int i = 0; i < s.size(); i++) {
        if (s[i] == '(')       st.push(')');
        else if (s[i] == '{')  st.push('}');
        else if (s[i] == '[')  st.push(']');
        else if (st.empty() || st.top() != s[i]) return false;
        else st.pop();
    }
    return st.empty();
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    string s = "({[]})";

    if (isValid(s)) {
        cout << "true" << endl;
    } else {
        cout << "false" << endl;
    }

    return 0;
}
```

## 删除字符串中的所有相邻重复项

给定一个字符串，反复删除两个相邻的相同字符，直到无法继续删除。例如输入 `"abbaca"`，删除 `"bb"` 后得到 `"aaca"`，再删除 `"aa"` 后得到 `"ca"`。

这本质是一个“相邻匹配消除”的问题，非常适合用栈解决：遍历字符串，每个字符与栈顶比较——相同则弹出栈顶（消除），不同则压入栈。最终栈中剩余字符就是答案。

这里没有显式使用 `std::stack`，而是用 `std::string` 直接作为栈——`push_back` 对应 push，`pop_back` 对应 pop，`back` 对应 top。这样做的好处是最终答案可以直接输出 string，不需要再倒一次栈。

代码：

```cpp
#include <bits/stdc++.h>
using namespace std;

string removeDuplicates(string s) {
    string ans;

    for (char c : s) {
        if (!ans.empty() && c == ans.back()) {
            ans.pop_back();
        } else {
            ans.push_back(c);
        }
    }

    return ans;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    string s = "abbaca";

    cout << removeDuplicates(s) << endl;

    return 0;
}
```
