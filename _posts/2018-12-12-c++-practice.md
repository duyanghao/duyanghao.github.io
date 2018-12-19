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

* 10 vector插入首部

```c++
vector<int> MyVector;

for (int i = 0; i < 10; i++)
    MyVector.insert(MyVector.begin(),i); // Inserts the value i at the start of the array 10 times.

for (vector<int>::iterator it=MyVector.begin(); it<MyVector.end(); it++)
    cout << *it; //Outputs the values of MyVector in order 
```

## c++编程思想

* 尽量用函数，避免代码臃肿
* 尽量用引用传参，避免开销
* STL尽可能使用常用的容器：vector、map、set等，完全足够大多数业务场景——越简单的容器用起来越不容易出错

## 业务开发&平台开发

* 业务开发节奏快；平台开发相对时间充裕（加班少）
* 业务开发对接'可见'的产品，需要对产品熟悉（有些时候，甚至比需求提出方更加熟悉）；平台开发一般是基础部门，可能大多数时候自己并不了解使用自己平台的业务
* 业务开发对知识的要求是：全而杂（核心是尽快把需求搞定，所以往往对某一个技术领域没有深钻）；平台开发则相反，目标是成为某个领域的'专家'
* 业务开发对接工程思维；平台开发对接专家思维
* 业务开发需求变更快；平台开发则相对稳定
* 业务开发大多写逻辑代码，对计算机底层包括整个架构的理解要求没有平台开发高
* 平台开发可以接触各种语言：go、scala、java、c/c++、perl、python、shell、c#等；业务开发则开发语言相对固定，通常只会使用1到2种语言
* 业务开发对需求驱动力要求更高，毕竟时间紧，任务急
* 平台开发对新知识的学习能力以及英文阅读能力要求更高（以前工作基本每天都要学习新的开源知识）

## Refs

* [Efficiently moving contents of std::unordered_set to std::vector](https://stackoverflow.com/questions/42519867/efficiently-moving-contents-of-stdunordered-set-to-stdvector)
* [C++中int、string等常见类型转换](https://www.cnblogs.com/gaobw/p/7070622.html)
* [convert-vector-set](https://www.techiedelight.com/convert-vector-set-cpp/)
* [How to get current time and date in C++?](https://stackoverflow.com/questions/997946/how-to-get-current-time-and-date-in-c)
* [Yesterday's date using c++](https://www.daniweb.com/programming/software-development/threads/506043/yesterday-s-date-using-c)
* [Insert things in the beginning of a vector](http://www.cplusplus.com/forum/beginner/60348/)