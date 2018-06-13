---
title: Python中的装饰器
date: 2017-12-05 16:49:15
thumbnail: /css/images/decorator.jpg
tags:
- 装饰器
category:
- Python
---

Python中的装饰器用的较为普遍，大致思路是在函数运行前后封装一些操作，以实现诸如如打印日志、统计运行时间、保存中间变量的效果。本文通过几个实例说明装饰器的基本用法和应用场景。

# 1.引子

浏览网上各种解释装饰器的文章，提到最多的就是斐波那契数列的计算，这里先给出基础的计算斐波那契数列的函数：

``` python
def fib(n):
    if n <= 2:
        return n - 1
    else:
        return fib(n - 1) + fib(n - 2)
```

以上代码采用递归的方式计算第n个斐波那契数，其实这种递归计算方法会产生很多重复计算，我们将fib(5)的计算拆解开：

```
             fib(5)
            /      \
          fib(4)    fib(3)
         /     \    /    \
    fib(3)  fib(2) fib(2) fib(1)
    /    \
fib(2)   fib(1)
```

从上面图中可以看到，fib(3)计算了2次，fib(2)计算了3次，fib(1)计算了2次，如果能将递归过程中的中间变量存储起来，就可以节省出很多时间，这里用装饰器的方法存储这些中间变量，首先给出代码：

``` python
def cache(f):
    cache_dict = {}

    @wraps(f)
    def _cache(n):
        if n in cache_dict.keys():
            return cache_dict[n]
        else:
            cache_dict[n] = f(n)
        return cache_dict[n]

    return _cache

@cache
def fib(n):
    if n <= 2:
        return n - 1
    else:
        return fib(n - 1) + fib(n - 2)
```

我们在装饰器中定义了一个全局的dict，用来存储第i个斐波那契值，每次计算fib(i)之前先去dict中查看是否已经缓存改值，如果缓存了直接从dict中取，否则计算fib(i)并写入dict中。

以上就实现了通过装饰器缓存部分变量，达到减少重复计算的目的，下面我们来了解一下装饰器的运行机制，以及变量的生命周期。

# 2.装饰器原理剖析

从上面斐波那契数列的例子可以看到，装饰器其实是一个接受函数做参数，返回值为函数的函数。笼统的可以概括成以下的形式：

``` python
def decorator(f):
	def _wrap(args):
		do somthing
		result = f(args)
		do somthing
		return result
	retun _wrap


@decorator
def foo(args):
	do somthing
```

返回的函数其实包括了要运行的函数，并在函数运行前后做了若干操作。那当我们调用foo(args)到底发生了什么呢？

当显示的调用foo(args)时，可以认为先执行了装饰器函数decorator(f)，装饰器函数返回了函数_wrap(args), 整体的调用顺序即是
decorator(f)(args)，为了验证这个的结论，我们将上面斐波那契数列的例子修改一下，执行下面的语句：


``` python
print fib(20)
print cache(fib)(20)

#output
4181
4181
```

可以看到两种输出方式结果是一致的， 从而验证了对于装饰器调用顺序的结论。为了更好的理解装饰器的调用顺序，这里对引子中的例子进行修改，再增加一层装饰器，如下：

``` python

def cache(f):
    cache_dict = {"test": "foo"}

    @wraps(f)
    def _cache(n):
        if n in cache_dict.keys():
            return cache_dict[n]
        else:
            cache_dict[n] = f(n)
        return cache_dict[n]

    return _cache


def record(f):
    @wraps(f)
    def _wrap(n):
        start_time = time.time()
        result = f(n)
        end_time = time.time()
        logger.info('f_name:%s, n:%s, cost_time:%s', f.__name__, n, end_time - start_time)
        return result

    return _wrap


@record
@cache
def fib(n):
    if n <= 2:
        return n - 1
    else:
        return fib(n - 1) + fib(n - 2)
```

可以看到增加了record装饰器，作用是记录函数运行时间，先调用一下fib(20),看看结果:

```
2017-12-05 21:03:12,115 [140735241039872] - [decorate_learn.py 32] INFO n:2, cost_time:9.53674316406e-07
2017-12-05 21:03:12,115 [140735241039872] - [decorate_learn.py 32] INFO n:1, cost_time:3.09944152832e-06
2017-12-05 21:03:12,115 [140735241039872] - [decorate_learn.py 32] INFO n:3, cost_time:0.000406980514526
2017-12-05 21:03:12,115 [140735241039872] - [decorate_learn.py 32] INFO n:2, cost_time:2.14576721191e-06
2017-12-05 21:03:12,116 [140735241039872] - [decorate_learn.py 32] INFO n:4, cost_time:0.000722885131836
2017-12-05 21:03:12,116 [140735241039872] - [decorate_learn.py 32] INFO n:3, cost_time:3.09944152832e-06
2017-12-05 21:03:12,116 [140735241039872] - [decorate_learn.py 32] INFO n:5, cost_time:0.00133514404297
3
```

可以看到每次调用fib(n)函数的时间都被打印出来，如上面对装饰器调用顺序的结论，这里同样跑一下record(cache(fib))(5),得到如下结果：

```
2017-12-05 21:09:35,869 [140735241039872] - [decorate_learn.py 32] INFO n:2, cost_time:2.86102294922e-06
2017-12-05 21:09:35,869 [140735241039872] - [decorate_learn.py 32] INFO n:1, cost_time:3.09944152832e-06
2017-12-05 21:09:35,869 [140735241039872] - [decorate_learn.py 32] INFO n:3, cost_time:0.000430107116699
2017-12-05 21:09:35,869 [140735241039872] - [decorate_learn.py 32] INFO n:2, cost_time:1.90734863281e-06
2017-12-05 21:09:35,870 [140735241039872] - [decorate_learn.py 32] INFO n:4, cost_time:0.000657081604004
2017-12-05 21:09:35,870 [140735241039872] - [decorate_learn.py 32] INFO n:3, cost_time:2.14576721191e-06
2017-12-05 21:09:35,870 [140735241039872] - [decorate_learn.py 32] INFO n:5, cost_time:0.00082802772522
2017-12-05 21:09:35,870 [140735241039872] - [decorate_learn.py 32] INFO n:5, cost_time:0.000908136367798
3
```


以上研究了装饰器调用函数的流程，下面我们看下装饰器中变量的生命周期。
注意到在斐波那契数列的例子中，定义了cache_dict字典，那该字典何时被创建，何时被销毁呢，为此我们做以下实验：

```python
import sys
import decorate_learn


for i in range(5):
    decorate_learn.fib(i + 1)

reload(decorate_learn)

for i in range(5):
    decorate_learn.fib(i + 1)
```

装饰器也稍作改变，每次调用的时候打印cache_dict
``` python
def cache(f):
    cache_dict = {"test": "foo"}

    @wraps(f)
    def _cache(n):
        logger.info('n:%s,cache_dict:%s', n, cache_dict)
        if n in cache_dict.keys():
            return cache_dict[n]
        else:
            cache_dict[n] = f(n)
        return cache_dict[n]

    return _cache

```


之所以这么做，是因为没有找到太好能够显示变量创建销毁的方法，所以每次调用装饰器的时候打印该变量，看下改变量的内容是否有被清空重建，
看下输出日志：

```
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:1,cache_dict:{'test': 'foo'}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:2,cache_dict:{'test': 'foo', 1: 0}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:3,cache_dict:{'test': 'foo', 1: 0, 2: 1}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:2,cache_dict:{'test': 'foo', 1: 0, 2: 1}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:1,cache_dict:{'test': 'foo', 1: 0, 2: 1}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:4,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:3,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:2,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:5,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1, 4: 2}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:4,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1, 4: 2}
2017-12-06 09:45:07,733 [140735241039872] - [decorate_learn.py 16] INFO n:3,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1, 4: 2}
2017-12-06 09:45:07,734 [140735241039872] - [decorate_learn.py 16] INFO n:1,cache_dict:{'test': 'foo'}
2017-12-06 09:45:07,734 [140735241039872] - [decorate_learn.py 16] INFO n:2,cache_dict:{'test': 'foo', 1: 0}
2017-12-06 09:45:07,735 [140735241039872] - [decorate_learn.py 16] INFO n:3,cache_dict:{'test': 'foo', 1: 0, 2: 1}
2017-12-06 09:45:07,735 [140735241039872] - [decorate_learn.py 16] INFO n:2,cache_dict:{'test': 'foo', 1: 0, 2: 1}
2017-12-06 09:45:07,735 [140735241039872] - [decorate_learn.py 16] INFO n:1,cache_dict:{'test': 'foo', 1: 0, 2: 1}
2017-12-06 09:45:07,735 [140735241039872] - [decorate_learn.py 16] INFO n:4,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1}
2017-12-06 09:45:07,736 [140735241039872] - [decorate_learn.py 16] INFO n:3,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1}
2017-12-06 09:45:07,738 [140735241039872] - [decorate_learn.py 16] INFO n:2,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1}
2017-12-06 09:45:07,738 [140735241039872] - [decorate_learn.py 16] INFO n:5,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1, 4: 2}
2017-12-06 09:45:07,739 [140735241039872] - [decorate_learn.py 16] INFO n:4,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1, 4: 2}
2017-12-06 09:45:07,739 [140735241039872] - [decorate_learn.py 16] INFO n:3,cache_dict:{'test': 'foo', 1: 0, 2: 1, 3: 1, 4: 2}
```

从日志的输出可以看到，一次程序执行过程中，装饰器中dict是和fib函数同时存在的，只有当主程序退出时，dict才会销毁。
以上我们研究了装饰器的写法和一些简单原理，下面给出一种使用类写装饰器的方法。

# 3.装饰器的另一种写法

装饰器除了函数式的写法，还可以封装成类，并重写__call__方法即可，还是以菲波那切数列为例，代码如下：

```python
import time
from functools import wraps


class cache(object):
    def __init__(self):
        self.cache_dict = {}

    def __call__(self, f):
        @wraps(f)
        def _wrap(n):
            print self.cache_dict
            if n in self.cache_dict.keys():
                return self.cache_dict[n]
            else:
                self.cache_dict[n] = f(n)
            return self.cache_dict[n]
        return _wrap


class record(object):
    def __init__(self):
        pass

    def __call__(self, f):
        @wraps(f)
        def _wrap(n):
            start_time = time.time()
            result = f(n)
            end_time = time.time()
            print 'f_name:%s, n:%s, cost_time:%s' % (f.__name__, n, end_time - start_time)
            return result
        return _wrap


@record()
@cache()
def fib(n):
    if n <= 2:
        return n - 1
    else:
        return fib(n - 1) + fib(n - 2)


@record()
@cache()
def foo(n):
    print "foo"

# print fib(1)
print fib(20)
foo(1)
```

代码里需要解释的一点是我们引入了functools.wraps，目的是保持函数的类型一致。

```python
#with wraps
fib(1)
print fib.__name__

#out
#fib

#without wraps
fib(1)
print fib.__name__

#out
#_wrap
```

从上面可以看到，加了wrap可以让函数保持原有的名字


# 总结
以上简单介绍了装饰器的实现方法和一些自己的小探究，笔而简之,以备阙忘。