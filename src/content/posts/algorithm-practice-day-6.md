---
title: <题外话>算法练习 Day 6：字符串
published: 2026-06-20
description: 反转字符串、替换数字、翻转单词
image: ""
tags: [数据结构]
category: 算法练习
draft: false
lang: zh-CN
---

## 今日复盘

主要学习字符串相关的几类操作：反转、替换、右旋和单词翻转。

字符串问题的核心思路大多可以归结为双指针和整体-局部的反转策略。双指针在字符串中同样适用，左右指针相向而行交换字符是最朴素的反转方式；而当涉及更复杂的场景（如局部反转、从后向前填充）时，双指针依然是最核心的工具。

反转类的题目有一个重要的技巧：**先整体反转，再局部反转**（或反过来）。这套思路可以统一解决右旋字符串和翻转单词等看似不同的问题，本质上都是通过两次反转来调整顺序。

替换类题目（如替换数字）的巧妙之处在于**从后向前填充**——先计算好扩容后的长度，再用双指针从尾部开始搬运，避免了从前向后填充时反复移动元素的开销。

最难的KMP算法明天再说，之前学DS的时候学过，但是基本忘光了。

## 反转字符串

最基础的反转字符串，使用左右双指针相向而行，交换字符即可。

代码：

```cpp
#include <vector>
#include <iostream>
using namespace std;

void reverseString(vector<char>& s) {
    int left = 0, right = s.size() - 1;

    while (left < right) {
        swap(s[left], s[right]);
        left++;
        right--;
    }
}

int main() {
    vector<char> s = {'h', 'e', 'l', 'l', 'o'};
    reverseString(s);
    for (char i : s) {
        cout << i << endl;
    }
    return 0;
}
```

## 反转字符串Ⅱ

给定一个字符串 s 和一个整数 k，每计数至 2k 个字符时，就反转这 2k 个字符中的前 k 个字符。如果剩余字符少于 k 个，则将剩余字符全部反转；如果剩余字符小于 2k 但大于或等于 k 个，则反转前 k 个字符，其余保持不变。

关键在于以 `2k` 为步长遍历，每次处理当前块的逻辑即可——要么反转前 k 个，要么反转剩余全部。

代码：

```cpp
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>  //包括reverse
using namespace std;

string reverseStr_sec(string s, int k) {
    for (int i = 0; i < s.size(); i += (2 * k)) {  //每隔2k个字符的前k个字符进行翻转
        if (i + k <= s.size()) {  //剩余字符小于2k但大于等于k个，则反转前k个字符
            reverse(s.begin() + i, s.begin() + i + k);
        } else {  //剩余字符少于k个，则全部反转
            reverse(s.begin() + i, s.end());
        }
    }
    return s;
}

int main() {
    string s1 = "abcdefg";
    s1 = reverseStr_sec(s1, 2);

    for (int i = 0; i < s1.size(); i++) {
        cout << s1[i] << ",";
    }
    return 0;
}
```

## 右旋字符串

将字符串的后 k 个字符移到前面。例如 `"ABCDEFG"` 右旋 2 位后变成 `"FGABCDE"`。

使用**三次反转**的技巧：先整体反转，再分别反转前 k 个和后 n-k 个。注意 `reverse` 的参数是左闭右开区间，因此反转前 k 个时区间是 `[begin, begin + k)`，反转后面部分时是 `[begin + k, end)`。

代码：

```cpp
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>  //包括reverse
using namespace std;

int main() {
    string s = "ABCDEFG";
    int k = 2;  //右旋倒数两个

    reverse(s.begin(), s.end());            // GFEDCBA
    reverse(s.begin(), s.begin() + k);      // FGEDCBA
    reverse(s.begin() + k, s.end());        // FGABCDE

    cout << s << "";
    return 0;
}
```

## 替换数字

给定一个字符串 s，它包含小写字母和数字字符，将字符串中的字母字符保持不变，而将每个数字字符替换为 `"number"`。

例如，输入 `"a1b2c3"`，输出 `"anumberbnumbercnumber"`。

这个问题的难点在于替换后字符串会变长（一个数字变成六个字符）。如果从前向后替换，每插入一个 `"number"` 都会导致后面的元素整体移动，效率低。

更好的做法是：先扫描一遍统计数字个数，计算扩容后的长度（每个数字多 5 个字符），然后使用双指针从后向前填充——旧指针从原字符串末尾开始，新指针从扩容后的末尾开始。遇到字母直接搬，遇到数字则逆序填入 `"number"`。

代码：

```cpp
#include <string>
#include <vector>
#include <iostream>
#include <algorithm>
#include <cctype>  //isdigit
using namespace std;

int main() {
    string s = "a5b2c8e";

    int count = 0;  //统计数字个数
    for (int i = 0; i < s.size(); i++) {
        if (isdigit(s[i])) {
            count++;
        }
    }
    int oldSize = s.size();
    s.resize(s.size() + count * 5);  //扩容（每个数字变成6个字符，比原来多5个）

    //双指针从后向前填充
    int oldIndex = oldSize - 1;
    int cur = s.size() - 1;

    while (oldIndex >= 0) {
        if (isdigit(s[oldIndex])) {
            s[cur--] = 'r';
            s[cur--] = 'e';
            s[cur--] = 'b';
            s[cur--] = 'm';
            s[cur--] = 'u';
            s[cur--] = 'n';
        } else {
            s[cur--] = s[oldIndex];
        }
        oldIndex--;
    }
    cout << s << endl;
    return 0;
}
```

## 翻转字符串里的单词

给定一个字符串，逐个翻转字符串中的每个单词。例如输入 `"I love you forver..."`，输出 `"...forver you love I"`。

思路和右旋字符串类似，采用**两次反转**的策略：先整体反转整个字符串，此时单词的顺序颠倒但每个单词本身也是逆序的；再遍历字符串找到每个单词的边界，将它们分别反转回来。用一个 `rear` 变量标记每个单词的起始位置，遇到空格时反转 `[rear, front)` 区间并将 `rear` 移到下一个单词开头。最后别忘了反转最后一个单词（它后面没有空格作为触发条件）。

代码：

```cpp
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>  //包括reverse
using namespace std;

int main() {
    string s = "I love you forver...";

    reverse(s.begin(), s.end());

    int length = s.length();
    int rear = 0;

    for (int front = 0; front < length; front++) {
        if (s[front] == ' ') {
            reverse(s.begin() + rear, s.begin() + front);
            rear = front + 1;
        }
    }
    reverse(s.begin() + rear, s.end());  //最后一个单词

    cout << s << "";
    return 0;
}
```
