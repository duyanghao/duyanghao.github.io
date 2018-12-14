---
layout: post
title: c++业务实践
date: 2018-12-12 16:33:31
category: 技术
tags: 业务开发
excerpt: c++业务开发常用技巧总结……
---

## 前言

本文列举出业务开发中常用的`c++`技巧，在本人用`c++`进行业务开发的过程中发现基本只需要使用这些特性就可以将`90%`的逻辑搞定……

## 基本语法注意知识

1. `c++`中可以用0表示false，用非0表示true
2. `c`打印`bool`，直接用`prinf %d`，如果为`false`则打印0；否则打印1
3. `c++` map类型可以直接使用没有key的value，value为default值
4. `c++` string convert int使用`atoi`; int convert string使用`std:to_string()`（添加`-std=c++11`参数）

## 常用技巧

1. 遍历vector

第一种方式：用下标遍历：
```c++
vector<int> v
for (size_t i = 0; i < v.size(); i++)
{
    v[i] ...
}
```

第二种方式：用迭代器遍历：
```c++
vector<int> v
for (vector<int>::iterator it = v.begin(); it != v.end(); ++it)
{
    *it ...
}
```

## 基本概念

## Refs

