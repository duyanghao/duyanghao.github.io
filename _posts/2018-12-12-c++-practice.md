---
layout: post
title: c++业务实践
date: 2018-12-12 16:33:31
category: 技术
tags: 业务开发
excerpt: c++业务开发常用技巧总结……
---

## 前言

本文列举出业务开发中常用的`c++`技巧和使用经验(不断补充)……

## 语法注意点

1. `c++`中可以用0表示false，用非0表示true
2. `c`打印`bool`，直接用`prinf %d`，如果为`false`则打印0；否则打印1
3. `c++` map类型可以直接使用没有key的value，value为default值
4. `c++` string convert int使用`atoi`; int convert string使用`std:to_string()`（添加`-std=c++11`参数）

## 常用技巧

* 1 遍历vector

第一种方式：用下标遍历：
```c++
vector<int> v;
for (size_t i = 0; i < v.size(); i++)
{
    v[i] ...
}
```

第二种方式：用迭代器遍历：
```c++
vector<int> v;
for (vector<int>::iterator it = v.begin(); it != v.end(); ++it)
{
    *it ...
}
```

* 2 vector遍历剔除

```c++
vector<int> v;
for (vector<int>::iterator it = v.begin(); it != v.end(); )
{
    if (xxx)
    {
        it = v.erase(it);
    }
    else
    {
        ++it;
    }
}
```

* 3 vector查找元素是否存在

```c++
vector<int> v;
vector<int>::iterator it = std::find(v.begin(), v.end(), xxx);
if (it != v.end())
{
    //处理存在情况
}
else
{
    //处理不存在情况
}
```

* 4 map查找某个元素是否存在

```c++
map<int, string> m;
map<int, string>::iterator it = m.find(xxx);
if (it != m.end())
{
    //处理存在情况
}
else
{
    //处理不存在情况
}
```

* 5 unordered_set转vector

```c++
std::unordered_set<int> u_s;
u_s.insert(xxx);
vector<int> v;
v.insert(v.end(), u_s.begin(), u_s.end());
```

* 6 vector转unordered_set

```c++
vector<int> v;
v.push_back(xxx);
std::unordered_set<int> u_s(v.begin(), v.end());
```

* 7 产生`1`——`10`随机化数字

```c++
srand (time(NULL));
j = 1 + (int) (10.0 * (rand() / (RAND_MAX + 1.0)));
...
```

> If you want to generate a random integer between 1 and 10, you should always do it by using high-order bits, as in

>                     j = 1 + (int) (10.0 * (rand() / (RAND_MAX + 1.0)));

>              and never by anything resembling

>                     j = 1 + (rand() % 10);

>              (which uses lower-order bits).

* 8 遍历map

```c++
for (std::map<int, string>::iterator it=m.begin(); it != m.end(); ++it)
    std::cout << it->first << " => " << it->second << '\n';
```

* 9 `string`去除最后一个字符

```c++
    string str;
    str = "123456";
    cout << str << endl;

    //方法一：使用substr()
    str = str.substr(0, str.length() - 1);
    cout << str << endl;

    //方法二：使用erase()
    str.erase(str.end() - 1);
    cout << str << endl;
```

## c++编程思想

* 尽量用函数，避免代码臃肿
* 尽量用引用传参，避免开销
* STL尽可能使用常用的容器：vector、map、set等，完全足够大多数业务场景——越简单的容器用起来越不容易出错

## Refs

* [Efficiently moving contents of std::unordered_set to std::vector](https://stackoverflow.com/questions/42519867/efficiently-moving-contents-of-stdunordered-set-to-stdvector)
* [C++中int、string等常见类型转换](https://www.cnblogs.com/gaobw/p/7070622.html)
* [convert-vector-set](https://www.techiedelight.com/convert-vector-set-cpp/)
* [How to get current time and date in C++?](https://stackoverflow.com/questions/997946/how-to-get-current-time-and-date-in-c)
* [Yesterday's date using c++](https://www.daniweb.com/programming/software-development/threads/506043/yesterday-s-date-using-c)