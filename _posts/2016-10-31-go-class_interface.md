---
layout: post
title: Go语言学习：类
date: 2016-10-31 22:49:51
category: 技术
tags: Golang
excerpt: Go语言类概述
---

本文概述了Go语言类的基本特征

## Go类

面向对象最重要的三个特征：封装、继承与多态，下面分别介绍：

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

编译报错，显示`ts.write undefined`，将write首字母改大写，运行成功！

同理，添加变量，变量小写访问编译报错，转大写后，运行成功！

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

**必须只填写类型，不能填写变量名，否则就是组合，不能通过变量名直接访问组合成员的方法和变量**

**Go在编译过程中会对组合类型逐项检查是否存在该变量和方法，若不存在或多于1个则报编译错误**

如下，在`Fz_str2`变量中也添加`Str`变量，运行报编译错误：

```sh
# go run main.go   
# command-line-arguments
./main.go:11: ambiguous selector ts.Str
```

#### 多态

用接口实现：某个类型的实例可以赋给它所实现的任意接口类型的变量，类型和接口是松耦合的

如下例子：

main.go函数：

```go
package main

import "fz"

func main(){
        var tst fz.Test_inter = new(fz.Sx_inter)
        tst.Print_hello()
        tst = new(fz.Sx2_inter)
        tst.Print_hello()
}
```

fz.go函数；

```go
package fz

import "fmt"

type Test_inter interface{
        Print_hello()
}

type Sx_inter struct{
        //...
}

type Sx2_inter struct{
        //...
}

func ( *Sx_inter) Print_hello(){
        fmt.Print("hello Sx_inter\n")
}

func ( *Sx2_inter) Print_hello(){
        fmt.Print("hello Sx2_inter\n")
}
```

运行:

```sh
# go run main.go 
hello Sx_inter
hello Sx2_inter
```

>An interface type specifies a method set called its interface. A variable of interface type can store a value of any type with a method set that is any superset of the interface. Such a type is said to implement the interface. The value of an uninitialized variable of interface type is nil.

也即接口是一些方法的集合（`method set`），接口类型的变量可以存储任何实现了该接口（**也即实现了接口声明的所有方法**）的类型变量，未初始化的接口变量值为`nil`

所以上面例子中`fz.(*Sx_inter)`指针类型变量可以赋值给`fz.Test_inter`接口类型变量

**注意**，如果将`fz.Sx_inter`值类型变量赋值给`fz.Test_inter`接口类型变量，如下：

```sh
var tst fz.Test_inter = fz.Sx_inter{} 
```

编译报错，如下：

```sh
# go run main.go 
# command-line-arguments
./main.go:6: cannot use fz.Sx_inter literal (type fz.Sx_inter) as type fz.Test_inter in assignment:
        fz.Sx_inter does not implement fz.Test_inter (Print_hello method has pointer receiver)
```

Go语言[`Method sets`](https://golang.org/ref/spec#Method_sets)规定如下：

>The method set of any other type T consists of all methods declared with receiver type T. The method set of the corresponding pointer type *T is the set of all methods declared with receiver *T or T (that is, it also contains the method set of T). 

简单的讲就是：指针类型（*T）包含`receiver`为 T 或者 *T的方法，而值类型（T）只包含`receiver`为 T 的方法。[effective go](https://golang.org/doc/effective_go.html#pointers_vs_values)中有这样的描述：

>We pass the address of a ByteSlice because only *ByteSlice satisfies io.Writer. The rule about pointers vs. values for receivers is that value methods can be invoked on pointers and values, but pointer methods can only be invoked on pointers.

具体原因参考[这里](https://golang.org/doc/faq#different_method_sets)

上面，我们实现了指针类型`fz.(*Sx_inter)`的方法`Print_hello`，而值类型`fz.Sx_inter`并没有实现该方法。也即`fz.(*Sx_inter)`指针类型实现了`fz.Test_inter`接口，可以对接口进行赋值；而`fz.Sx_inter`值类型并没有实现`fz.Test_inter`接口，无法对接口进行赋值

**注意：这里的描述有一个上下文，就是给接口赋值。除此之外，不管是值类型还是指针类型，都实现了`receiver`为 T 和 *T的方法**

## 补充

#### [值方法与指针方法](https://golang.org/doc/faq#methods_on_values_or_pointers)

值方法（value method，receiver为value）与指针方法（pointer method，receiver与pointer）

```go
func (s *MyStruct) pointerMethod() { } // method on pointer
func (s MyStruct)  valueMethod()   { } // method on value
```

Should I define methods on values or pointers?

>For programmers unaccustomed to pointers, the distinction between these two examples can be confusing, but the situation is actually very simple. When defining a method on a type, the receiver (s in the above examples) behaves exactly as if it were an argument to the method. Whether to define the receiver as a value or as a pointer is the same question, then, as whether a function argument should be a value or a pointer. There are several considerations.

>First, and most important, does the method need to modify the receiver? If it does, the receiver must be a pointer. (Slices and maps act as references, so their story is a little more subtle, but for instance to change the length of a slice in a method the receiver must still be a pointer.) In the examples above, if pointerMethod modifies the fields of s, the caller will see those changes, but valueMethod is called with a copy of the caller's argument (that's the definition of passing a value), so changes it makes will be invisible to the caller.

>By the way, pointer receivers are identical to the situation in Java, although in Java the pointers are hidden under the covers; it's Go's value receivers that are unusual.

>Second is the consideration of efficiency. If the receiver is large, a big struct for instance, it will be much cheaper to use a pointer receiver.

>Next is consistency. If some of the methods of the type must have pointer receivers, the rest should too, so the method set is consistent regardless of how the type is used. See the section on method sets for details.

>For types such as basic types, slices, and small structs, a value receiver is very cheap so unless the semantics of the method requires a pointer, a value receiver is efficient and clear.

那么什么时候用值方法，什么时候用指针方法呢？主要考虑以下一些因素：

* **如果方法要修改`receiver`，那么必须是指针方法**
* **指针方法是常用的，而值方法是不常用的**
* **考虑到效率，指针方法更好（传递 值类型vs指针类型）**
* **一致性，要么全部都是指针方法，要么全部都是值方法**
* **对于一些基本类型、切片、或者小的结构体，使用value receiver效率会更高一些**

总结：除非必须使用指针方法，一般情况下使用值方法效率更高，也更清晰

## 参考

* [interface](http://hustcat.github.io/interface/)
* [Effective Go](https://golang.org/doc/effective_go.html)
* [Interface types](https://golang.org/ref/spec#Interface_types)
* [Go by Example: Interfaces](https://gobyexample.com/interfaces)
* [Cannot define new methods on non-local type](http://stackoverflow.com/questions/28800672/how-to-add-new-methods-to-an-existing-type-in-go)