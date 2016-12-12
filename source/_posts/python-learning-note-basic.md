---
title: Python 学习笔记 - 基础篇
date: 2016-12-12 17:31:58
tags: [Python]
---

本篇文章是学习[廖雪峰 Python 教程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000)过程中自己记录的一些笔记，很多内容都只是针对我自己现有知识的补充，所以可能不具备普遍的参考意义。

如果对原教程有兴趣，可以直接上[廖雪峰老师的网站](http://www.liaoxuefeng.com)进行学习。

<!-- more -->

## 输入输出

使用 `print` 函数进行输出：

```python
print('hello, world')
print('The quick brown fox', 'jumps over', 'the lazy dog')
print('100 + 200 =', 100 + 200)
```

使用 `input` 函数进行输入：

```python
name = input('Please enter your name: ')
print('hello,', name)
```

## 数据类型和变量

Python 中有如下几种数据类型：
* 整数
* 浮点数
* 字符串
    - 可以使用单引号 `''` 也可以使用双引号 `""`
    - 使用 `r''` 形式可以防止字符串内部转义
    - 使用 `'''...'''` 三个单引号可以输入多行文字
* 布尔值
    - 只有两个值，True 和 False
    - 可以使用 and、or 和 not 运算
* 空值
    - 使用 None 表示

Python 是动态类型的语言，声明变量时不需要指定类型，可以随时给变量赋不同类型的值：

```python
a = 123
a = 'ABC'
```

Python 中没有声明常量的语法，一般使用大写字母来表示常量：

```python
PI = 3.14159265359
```

Python 中的两种除法：
* `/` 为浮点除法，得出的结果是浮点数
* `//` 为向下整除，得出的结果是整数，并且是向下取整的

## 字符串和编码

编码的发展：

* ASCII - 使用一个字节存储，不能用于处理非英文国家的编码
* Unicode - 还在发展中，正常使用两个字节表示一个字符
* UTF-8 - 对 Unicode 的变种，可以把一个 Unicode 字符根据不同的数字大小编码成 1-6 个字节

现代计算机系统通用的字符编码工作方式：在计算机内存中，统一使用 Unicode 编码，当需要保存到硬盘或者需要传输的时候，就转换为 UTF-8 编码。

在 Python3 当中，字符串是以 Unicode 编码的，即 Python3 的字符串支持多语言。

使用 `ord()` 函数可以获取字符的整数表示，使用 `chr()` 函数可以把编码转换为对应的字符：

```python
>>> ord('A')
65
>>> ord('中')
20013
>>> chr(66)
'B'
>>> chr(25991)
'文'
```

Python 的字符串类型是 `str`, 在内存中以 Unicode 表示，一个字符对应若干个字节。

如果要在网络上传输，或者保存到磁盘上，需要把 `str` 变为以字节为单位的 `bytes`，可以使用 `encode()` 方法将 `str` 编码为指定字符集的 `bytes`：

```python
>>> 'ABC'.encode('ascii')
b'ABC'
>>> '中文'.encode('utf-8')
b'\xe4\xb8\xad\xe6\x96\x87'
```

反之，如果从网络或磁盘上读取了字节流，需要使用 `decode()` 方法转换为 `str` 类型：

```python
>>> b'ABC'.decode('ascii')
'ABC'
>>> b'\xe4\xb8\xad\xe6\x96\x87'.decode('utf-8')
'中文'
```

可以使用 `len()` 函数计算 `str` 字符数，如果要计算 `bytes` 字符数，需要先将 `str` 编码为 `bytes` 类型：

```python
>>> len('中文')
2
>>> len('中文'.encode('utf-8'))
6
```

当 Python 源代码中包含中文时，为了让解释器按 UTF-8 编码进行读取，我们需要在源文件开头写上这两行：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
```

Python 中使用 % 对字符串进行格式化，占位符与 C 语言是一致的：

```python
>>> 'Hello, %s' % 'world'
'Hello, world'
>>> 'Hi, %s, you have $%d.' % ('Michael', 1000000)
'Hi, Michael, you have $1000000.'
```

## 使用 list 和 tuple

list 是一种有序的集合，可以随时添加和删除其中的元素。

可以使用索引来获取指定位置的元素，也可以使用负数索引从最后一个元素进行获取。当索引越界时，会引发一个 IndexError 异常。

list 常用的方法：

* len() 获取 list 元素个数
* append() 追加元素到末尾
* insert() 把元素插入到指定的位置
* pop() 删除末尾元素，pop(i) 删除指定位置元素

tuple 也是一种有序列表，与 list 不同的是，tuple 是不可变的。

tuple 的元素必须在定义时就确定下来，定义一个元素的 tuple 方法如下：

```python
>>> t = (1,)
>>> t
(1,)
```

## 条件判断

Python 的条件判断与其它语言的基本一致，只有两点需要注意：
1. 判断条件不使用括号，需要在判断表达式后面使用冒号
2. 代码块不使用花括号，直接使用缩近

`input` 函数的返回值是 `str` 类型，如果需要获取整数类型，需要使用 `int()` 函数进行类型转换。

## 循环

Python 中有两种循环形式：
* for-in
* while

在 Python 当中没有普通形式的循环，可以使用 `for-in` 和 `range()` 方法来进行模拟：

```python
for i in range(10):
    // ...
```

同样地，在 Python 当中也没有 `do-while` 循环。

## 使用 dict 和 set

dict 即字典，使用方法与 Swift 中的字典大同小异。

字典的初始化：

```python
d = {'Michael': 95, 'Bob': 75, 'Tracy': 85}
```

当使用 key 对字典进行索引时，如果 key 不存在则 dict 会抛出错误。

可以使用 `in` 关键字判断 key 是否存在：

```python
'Thomas' in d
False
```

也可以使用 dict 提供的 `set` 方法，如果 key 不存在，则返回 None，或者自己指定的 value：

```python
>>> d.get('Thomas')
>>> d.get('Thomas', -1)
-1
```

要删除 key 可以使用 `pop(key)` 方法，对应的 value 也会从 dict 中删除。

要正确使用 dict 必须保证它的 key 是**不可变对象**，Python 中的字符串、整数等都是不可变对象，可以放心使用。

set 也是一组 key 的集合，但是它不存储 value。set 的特点就是存储的 key 不会重复，同时是无序的。

要创建 set 需要使用 list 作为输入集合：

```python
>>> s = set([1, 2, 3])
>>> s
{1, 2, 3}
```

可以使用 `add(key)` 和 `remove(key)` 向 set 添加和删除元素。

set 也可以进行取合集和并集操作：

```python
>>> s1 = set([1, 2, 3])
>>> s2 = set([2, 3, 4])
>>> s1 & s2
{2, 3}
>>> s1 | s2
{1, 2, 3, 4}
```

## 函数

Python 内置函数官方文档：[https://docs.python.org/3/library/functions.html](https://docs.python.org/3/library/functions.html)

也可以使用 `help(<function>)` 进行查询。

Python 使用 `def` 关键字定义函数，定义时不需要指定返回值类型，并且如果在函数体内没有使用 return 语句进行返回，则函数到达函数尾时默认返回 None。

Python 可以使用 tuple 模拟返回多个值：

```python
import math

def move(x, y, step, angle=0):
    nx = x + step * math.cos(angle)
    ny = y - step * math.sin(angle)
    return nx, ny
```

### 函数参数

Python 的函数定义很简单，但灵活度却非常大。除了正常定义的必选参数外，还可以使用默认参数、可变参数和关键字参数。

使用默认参数比较简单，需要注意两点：
1. 必选参数在前，默认参数在后，否则解释器会报错
2. 当函数有多个参数时，把变化大的参数放在前面，变化小的参数放后面。变化小的参数可以作为默认参数。

默认参数的调用方法：可以按顺序提供默认参数，也可以不按顺序但是提供参数名：

```python
def enroll(name, gender, age=6, city='Beijing'):
    print('name:', name)
    print('gender:', gender)
    print('age:', age)
    print('city:', city)

enroll('Bob', 'M', 7)
enroll('Adam', 'M', city='Tianjin')
```

定义默认参数还有最重要的一点：默认参数必须指向不变对象！

**可变参数**的定义：

```python
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
```

这种形式在函数内部接收到的参数是一个 tuple，调用该函数时可以传入任意个参数，包括 0 个参数：

```python
>>> calc(1, 2)
5
>>> calc()
0
```

如果已经有一个 list 或 tuple，可以使用如下方法调用可变参数方法：

```python
>>> nums = [1, 2, 3]
>>> calc(*nums)
14
```

**关键字参数**允许传入 0 个或任意个含参数名的参数，这个关键字在函数内部自动组装为一个 dict，定义形式：

```python
def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)
```

这时可以只传入必传参数，也可以传入任意个数的关键字参数：

```python
>>> person('Michael', 30)
name: Michael age: 30 other: {}
>>> person('Adam', 45, gender='M', job='Engineer')
name: Adam age: 45 other: {'gender': 'M', 'job': 'Engineer'}
```

如果已经有一个 dict 了，可以使用如下方式调用：

```python
>>> extra = {'city': 'Beijing', 'job': 'Engineer'}
>>> person('Jack', 24, **extra)
name: Jack age: 24 other: {'city': 'Beijing', 'job': 'Engineer'}
```

如果要限制关键字参数的名字，就可以用命名关键字参数，定义方式如下：

```python
def person(name, age, *, city, job):
    print(name, age, city, job)
```

和关键字参数**kw不同，命名关键字参数需要一个特殊分隔符*，*后面的参数被视为命名关键字参数。

调用方式如下：

```python
>>> person('Jack', 24, city='Beijing', job='Engineer')
Jack 24 Beijing Engineer
```

如果函数定义中已经有了一个可变参数，后面跟着的命名关键字参数就不再需要一个特殊分隔符*了：

```python
def person(name, age, *args, city, job):
    print(name, age, args, city, job)
```

命名关键字参数必须传入参数名，这和位置参数不同。如果没有传入参数名，调用将报错。

命名关键字可以有缺省值，从而简化调用。

在Python中定义函数，可以用必选参数、默认参数、可变参数、关键字参数和命名关键字参数，这5种参数都可以组合使用。但是请注意，参数定义的顺序必须是：必选参数、默认参数、可变参数、命名关键字参数和关键字参数。
