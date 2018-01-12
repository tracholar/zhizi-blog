---
title: python yield 关键字
date: 2018-01-12 16:02:23
tags: python
categories: python
---


要理解什么yield，你必须了解什么是 generator。

## Iterables
当你创建一个列表，你可以逐个读取它的项目。逐个读取它的项目称为迭代：

```python
>>> mylist = [1, 2, 3]
>>> for i in mylist:
...    print(i)
1
2
3
```

mylist是一个迭代。当你使用列表推导，你创建一个列表，和一个可迭代的：

```python
>>> mylist = [x*x for x in range(3)]
>>> for i in mylist:
...    print(i)
0
1
4
```

你可以使用`for... in...`的所有东西都是可迭代的; lists，strings，file ...

这些迭代器很方便，因为您可以随心所欲地读取它们，但是将所有值存储在内存中，而且当有很多值时，并不总是您想要的值。

## generator
生成器(generator)是迭代器，一种迭代器，只能迭代一次。generator不会将所有的值存储在内存中，它们会立即生成值：

```python
>>> mygenerator = (x*x for x in range(3))
>>> for i in mygenerator:
...    print(i)
0
1
4
```

除了你用来`()`代替`[]`之外，它是一样的。但是，由于generator只能使用一次，所以不能再执行`for i in mygenerator`多次：计算0，然后忘记计算1，逐个计算4。

## yield
yield是一个像return的关键字，函数将返回一个生成器。

```python
>>> def createGenerator():
...    mylist = range(3)
...    for i in mylist:
...        yield i*i
...
>>> mygenerator = createGenerator() # create a generator
>>> print(mygenerator) # mygenerator is an object!
<generator object createGenerator at 0xb7555c34>
>>> for i in mygenerator:
...     print(i)
0
1
4
```

这是一个无用的例子，但是当你知道你的函数将会返回一大堆你只需要读取一次的值的时候，它是很方便的。

要掌握yield，你必须明白当你调用函数时，你写在函数体中的代码不会运行。该函数只返回生成器对象，这有点棘手:-)

然后，您的代码将在每次使用for时运行生成器。

困难的部分：

第一次for调用由你的函数创建的generator对象时，它会从你的函数开始直到碰到它yield，然后返回第一个循环的值。然后，每个其他的调用将运行你已经写在函数中的循环一次，并返回下一个值，直到没有值返回。

一旦函数运行，生成器被认为是空的，但不再被yield触发。这可能是因为循环已经结束，或者因为你不再满足"if/else"了。

生成器的高级用法：

## 控制生成器

```python
>>> class Bank(): # let's create a bank, building ATMs
...    crisis = False
...    def create_atm(self):
...        while not self.crisis:
...            yield "$100"
>>> hsbc = Bank() # when everything's ok the ATM gives you as much as you want
>>> corner_street_atm = hsbc.create_atm()
>>> print(corner_street_atm.next())
$100
>>> print(corner_street_atm.next())
$100
>>> print([corner_street_atm.next() for cash in range(5)])
['$100', '$100', '$100', '$100', '$100']
>>> hsbc.crisis = True # crisis is coming, no more money!
>>> print(corner_street_atm.next())
<type 'exceptions.StopIteration'>
>>> wall_street_atm = hsbc.create_atm() # it's even true for new ATMs
>>> print(wall_street_atm.next())
<type 'exceptions.StopIteration'>
>>> hsbc.crisis = False # trouble is, even post-crisis the ATM remains empty
>>> print(corner_street_atm.next())
<type 'exceptions.StopIteration'>
>>> brand_new_atm = hsbc.create_atm() # build a new one to get back in business
>>> for cash in brand_new_atm:
...    print cash
$100
$100
$100
$100
$100
$100
$100
$100
$100
...
```

注意：对于Python3使用`print(corner_street_atm.__next__())`或`print(next(corner_street_atm))`

它可以用于控制对资源的访问等各种功能。

## Itertools
itertools模块包含特殊的函数来操作iterables。是否希望复制一个生成器？链接两个生成器？Map / Zip而不创建另一个列表？

然后，只需`import itertools`。

一个例子，让我们看看四匹马的可能的到达顺序：

```python
>>> horses = [1, 2, 3, 4]
>>> races = itertools.permutations(horses)
>>> print(races)
<itertools.permutations object at 0xb754f1dc>
>>> print(list(itertools.permutations(horses)))
[(1, 2, 3, 4),
 (1, 2, 4, 3),
 (1, 3, 2, 4),
 (1, 3, 4, 2),
 (1, 4, 2, 3),
 (1, 4, 3, 2),
 (2, 1, 3, 4),
 (2, 1, 4, 3),
 (2, 3, 1, 4),
 (2, 3, 4, 1),
 (2, 4, 1, 3),
 (2, 4, 3, 1),
 (3, 1, 2, 4),
 (3, 1, 4, 2),
 (3, 2, 1, 4),
 (3, 2, 4, 1),
 (3, 4, 1, 2),
 (3, 4, 2, 1),
 (4, 1, 2, 3),
 (4, 1, 3, 2),
 (4, 2, 1, 3),
 (4, 2, 3, 1),
 (4, 3, 1, 2),
 (4, 3, 2, 1)]
```

## 了解迭代的内在机制
迭代是一个隐式迭代（实现`__iter__()`方法）和迭代（实现`__next__()`方法）的过程。Iterables是可以从中获取迭代器的任何对象。迭代器是可以迭代迭代的对象。
