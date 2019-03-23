---
title: Python闭包和go闭包
date: 2019-03-22 16:49:15
thumbnail: /css/images/closure.jpg
tags:
- 闭包
category:
- Python
- Go
---

最近学习golang，发现golang里面也有闭包的概念，这里和python比较一下，并有些新的体会，记下来

# 1.Python 闭包

闭包：引用了自由变量的函数，从下面这个例子看出，decorator的返回值_wrap引用了两个自由变量f和cache
通过__closure__可以一探究竟，输出结果可以看出__closure__包含两个元素，一个是function，另外一个是cache，


```python
def decorator(f):
    cache = []
    def _wrap(*args, **kwargs):
        cache.append(1)
        print(cache)
        f(*args, **kwargs)

    return _wrap

# @decorator
def foo():
    print("foo")

if __name__ == "__main__": 

    a = decorator(foo)
    a()
    print(a.__closure__)
    print(a.__closure__[1].cell_contents)
    print(a.__closure__[0].cell_contents)

# output

#foo
#(<cell at 0x1094cd198: list object at 0x10acea248>, <cell at 0x1094cd4c8: function object at 0x107879268>)
#<function foo at 0x107879268>
#[1]
```


之前写过一篇对于闭包中引用变量的生命周期，这次做了个实验，先看结果

```python
def decorator(f):
    cache = []

    def _wrap(*args, **kwargs):
        cache.append(1)
        print(cache)
        f(*args, **kwargs)

    return _wrap


# @decorator
def foo():
    print("foo")


if __name__ == "__main__":
    a = decorator(foo)
    a()
    a()
    print(a.__closure__)
    print(a.__closure__[1].cell_contents)
    print(a.__closure__[0].cell_contents)

    b = decorator(foo)
    b()
    b()
    print(b.__closure__)
    print(b.__closure__[1].cell_contents)
    print(b.__closure__[0].cell_contents)

# output
# [1]
# foo
# [1, 1]
# foo
# (<cell at 0x10eb34198: list object at 0x11a657248>, <cell at 0x10eb344c8: function object at 0x10cee0268>)
# <function foo at 0x10cee0268>
# [1, 1]
# [1]
# foo
# [1, 1]
# foo
# (<cell at 0x10eb34a98: list object at 0x112abb188>, <cell at 0x10eb34af8: function object at 0x10cee0268>)
# <function foo at 0x10cee0268>
# [1, 1]
```

可以看到闭包的自由变量的作用域对于每个函数是独立的，即a和b对于f和cache的引用是独立的


# 2.Go闭包

go的闭包和python基本一样，看个例子

```golang
package main

import "fmt"

func decorator(f func()) func(){
    var cache []string
    cache = append(cache, "1")
    _wrap := func() {
        cache = append(cache, "foo")
        fmt.Printf("%s\n", cache)
        f()
    }
    return _wrap
}

func foo() {
    fmt.Println("foo")
}

func main() {
  a := decorator(foo)
  a()
  a()
  b := decorator(foo)
  b()
  b()
}
// output
// [1 foo]
// foo
// [1 foo foo]
// foo
// [1 foo]
// foo
// [1 foo foo]
//foo
```
可以看到对于a和b对于自用变量的引用也是独立的，互不影响