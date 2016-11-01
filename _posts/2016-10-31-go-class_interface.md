---
layout: post
title: Go类与接口
date: 2016-10-31 22:49:51
category: 技术
tags: Go
excerpt: Go类与接口的使用总结
---

本文总结了Go类和接口的使用

### Go类

#### 封装

Go将封装简化为2层（不光是类，对Go语言中任何标识符都生效）：

* 包范围内：通过标识符首字母小写，变量和方法都只能在包内可见

* 可导出的：通过标识符首字母大写，变量和方法在包以外也可见

例子：

目录结构：
```sh
.
├── main.go
└── src
    └── fz
        └── fz.go
```

main.go函数:

```go
package main

import "fz"

func main(){
        ts := fz.Fz_str{}
        ts.Read()
        ts.write()
}
```

fz.go函数：

```go
package fz

type Fz_str struct{
        //...
}

func ( *Fz_str) Read(){
        //...
}

func ( *Fz_str) write(){
        //...
}
```

运行：

```sh
go run main.go      
# command-line-arguments
./main.go:8: ts.write undefined (cannot refer to unexported field or method fz.(*Fz_str)."".write)
```

编译报错，显示`ts.write undefined`，将write改首字母改大写，运行成功！

同理，变量小写编译报错，转大写后，运行成功！

#### 继承

Go不直接支持继承，不过可以通过组合来间接实现：内嵌一个或多个（多重继承）其它类型（包含变量和方法）

如下例子：

main.go函数；

```go
package main

import "fz"
import "fmt"

func main(){
  ts := fz.Fz_jc{fz.Fz_str{"str1"},fz.Fz_str2{"str2"}}
  ts.Read()
  ts.Write()
  ts.Read2()
  fmt.Printf("%s\n", ts.Str)
  fmt.Printf("%s\n", ts.Str2)
}
```

fz.go函数:

```go
package fz

import "fmt"

type int_fz interface{
        Read()
        Write()
}

type Fz_str struct{
        Str string
}

type Fz_str2 struct{
        Str2 string
}

type Fz_jc struct{
        Fz_str
        Fz_str2
}

func ( *Fz_str) Read(){
        fmt.Print("hello")
        //...
}

func ( *Fz_str) Write(){
        fmt.Print("hello")
        //...
}

func (s *Fz_str2) Read2(){
        fmt.Printf("%s\n", s.Str2)
}
```

运行：

```sh
#go run main.go 
hellohellostr2
str1
str2
```

**注意**：若将`Fz_jc`结构体修改如下：

```go
type Fz_jc struct{
  A Fz_str
  B Fz_str2
}
```

则编译出错，如下：

```sh
# go run main.go 
# command-line-arguments
./main.go:8: ts.Read undefined (type fz.Fz_jc has no field or method Read)
./main.go:9: ts.Write undefined (type fz.Fz_jc has no field or method Write)
./main.go:10: ts.Read2 undefined (type fz.Fz_jc has no field or method Read2)
./main.go:11: ts.Str undefined (type fz.Fz_jc has no field or method Str)
./main.go:12: ts.Str2 undefined (type fz.Fz_jc has no field or method Str2)
```

**必须只填写类型，不能填写名变量名，否则就是组合了**

**Go在编译过程中会对组合类型逐项检查是否存在该变量和方法，若不存在或多于1个则报编译错误**

如下，在`Fz_str2`变量中也添加`Str`变量，运行报编译错误：

```sh
# go run main.go   
# command-line-arguments
./main.go:11: ambiguous selector ts.Str
```

#### 多态



### Go接口
