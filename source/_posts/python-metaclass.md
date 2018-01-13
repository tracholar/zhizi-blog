---
title: python metaclass
date: 2018-01-12 21:49:19
tags:
categories: python
---

## 简介
元类是一个类的类。就像一个类定义一个类的实例的行为一样，元类定义了一个类的行为。一个类是一个元类的一个实例。

![元类图](https://i.stack.imgur.com/QQ0OK.png)

在Python中，你可以为元类使用任意的可调参数，但更有用的方法实际上是将它自己变成一个实际的类。type是Python中常用的元类。type它本身就是一个类，而且是它自己的类型。你将无法在Python中重新创建像type一样的东西，但是Python会作弊。要在Python中创建自己的元类，你只要type的子类。

元类最常用作类工厂。就像通过调用类创建类的实例一样，Python通过调用元类来创建一个新类（当它执行'class'语句时）。结合常规__init__和__new__方法，元类因此允许你在创建一个类时做“额外的事情”，比如用一些注册表来注册新类，或者甚至用其他东西来完全替换类。

当class语句被执行时，Python首先class以正常的代码块的形式执行语句的主体。由此产生的命名空间（一个字典）保存待分类的属性。元类是通过查看将被类（元类继承的）的基类，要被类__metaclass__（如果有的话）或__metaclass__全局变量的属性来确定的。元类然后调用类的名称，基类和属性来实例化它。

然而，元类实际上定义了一个类的类型，而不仅仅是一个工厂，所以你可以做更多的事情。例如，您可以在元类上定义常规方法。这些元类方法就像类方法，因为它们可以在没有实例的情况下在类上调用，但是它们也不像类方法，因为它们不能在类的实例上调用。type.__subclasses__()是type元类的一个方法的一个例子。您还可以定义正常的“魔力”的方法，如__add__，__iter__和__getattr__，执行或更改类的行为。

以下是一些小部分的汇总示例：

```python
def make_hook(f):
    """Decorator to turn 'foo' method into '__foo__'"""
    f.is_hook = 1
    return f

class MyType(type):
    def __new__(cls, name, bases, attrs):

        if name.startswith('None'):
            return None

        # Go over attributes and see if they should be renamed.
        newattrs = {}
        for attrname, attrvalue in attrs.iteritems():
            if getattr(attrvalue, 'is_hook', 0):
                newattrs['__%s__' % attrname] = attrvalue
            else:
                newattrs[attrname] = attrvalue

        return super(MyType, cls).__new__(cls, name, bases, newattrs)

    def __init__(self, name, bases, attrs):
        super(MyType, self).__init__(name, bases, attrs)

        # classregistry.register(self, self.interfaces)
        print "Would register class %s now." % self

    def __add__(self, other):
        class AutoClass(self, other):
            pass
        return AutoClass
        # Alternatively, to autogenerate the classname as well as the class:
        # return type(self.__name__ + other.__name__, (self, other), {})

    def unregister(self):
        # classregistry.unregister(self)
        print "Would unregister class %s now." % self

class MyObject:
    __metaclass__ = MyType


class NoneSample(MyObject):
    pass

# Will print "NoneType None"
print type(NoneSample), repr(NoneSample)

class Example(MyObject):
    def __init__(self, value):
        self.value = value
    @make_hook
    def add(self, other):
        return self.__class__(self.value + other.value)

# Will unregister the class
Example.unregister()

inst = Example(10)
# Will fail with an AttributeError
#inst.unregister()

print inst + inst
class Sibling(MyObject):
    pass

ExampleSibling = Example + Sibling
# ExampleSibling is now a subclass of both Example and Sibling (with no
# content of its own) although it will believe it's called 'AutoClass'
print ExampleSibling
print ExampleSibling.__mro__
```



## 类作为对象
在理解元类之前，你需要掌握Python中的类。而且Python从Smalltalk语言中借用了一个非常奇怪的概念。

在大多数语言中，类只是描述如何生成对象的代码段。在Python中也是如此：

```python
>>> class ObjectCreator(object):
...       pass
...

>>> my_object = ObjectCreator()
>>> print(my_object)
<__main__.ObjectCreator object at 0x8974f2c>
```

但是类不仅仅是Python。类也是对象。


只要你使用关键字class，Python就执行它并创建一个OBJECT。指令

```python
>>> class ObjectCreator(object):
...       pass
...
```

在内存中创建名为“ObjectCreator”的对象。

这个对象（类）本身能够创建对象（实例），这就是为什么它是一个类。

但是，这仍然是一个客体，因此：

你可以把它分配给一个变量
你可以复制它
你可以添加属性
您可以将其作为函数参数传递
例如：

```python
>>> print(ObjectCreator) # you can print a class because it's an object
<class '__main__.ObjectCreator'>
>>> def echo(o):
...       print(o)
...
>>> echo(ObjectCreator) # you can pass a class as a parameter
<class '__main__.ObjectCreator'>
>>> print(hasattr(ObjectCreator, 'new_attribute'))
False
>>> ObjectCreator.new_attribute = 'foo' # you can add attributes to a class
>>> print(hasattr(ObjectCreator, 'new_attribute'))
True
>>> print(ObjectCreator.new_attribute)
foo
>>> ObjectCreatorMirror = ObjectCreator # you can assign a class to a variable
>>> print(ObjectCreatorMirror.new_attribute)
foo
>>> print(ObjectCreatorMirror())
<__main__.ObjectCreator object at 0x8997b4c>
```

## 动态创建类
由于类是对象，因此可以像任何对象一样快速创建它们。

首先，您可以使用class以下方法在函数中创建一个类：

```python
>>> def choose_class(name):
...     if name == 'foo':
...         class Foo(object):
...             pass
...         return Foo # return the class, not an instance
...     else:
...         class Bar(object):
...             pass
...         return Bar
...     
>>> MyClass = choose_class('foo')
>>> print(MyClass) # the function returns a class, not an instance
<class '__main__.Foo'>
>>> print(MyClass()) # you can create an object from this class
<__main__.Foo object at 0x89c6d4c>
```

但是它不是那么有活力，因为你还得自己写全班。

由于类是对象，它们必须由某种东西产生。

当你使用class关键字时，Python会自动创建这个对象。但是与Python中的大多数事情一样，它给了你一个手动的方法。

记得功能type？良好的旧功能，让你知道什么类型的对象是：

```python
>>> print(type(1))
<type 'int'>
>>> print(type("1"))
<type 'str'>
>>> print(type(ObjectCreator))
<type 'type'>
>>> print(type(ObjectCreator()))
<class '__main__.ObjectCreator'>
```

那么，type有一个完全不同的能力，它也可以在飞行中创建类。type可以把一个类的描述作为参数，并返回一个类。

（我知道，根据你传递给它的参数，相同的函数可能有两种完全不同的用法，这很愚蠢，这是由于Python中的向后兼容性问题）

type 这样工作：

type(name of the class,
     tuple of the parent class (for inheritance, can be empty),
     dictionary containing attributes names and values)
例如：

```python
>>> class MyShinyClass(object):
...       pass
```

可以这样手动创建：

```python
>>> MyShinyClass = type('MyShinyClass', (), {}) # returns a class object
>>> print(MyShinyClass)
<class '__main__.MyShinyClass'>
>>> print(MyShinyClass()) # create an instance with the class
<__main__.MyShinyClass object at 0x8997cec>
```

你会注意到我们使用“MyShinyClass”作为类的名字，并作为变量来保存类的引用。他们可以是不同的，但没有理由使事情复杂化。

type接受一个字典来定义类的属性。所以：

```python
>>> class Foo(object):
...       bar = True
```

可以翻译成：

```python
>>> Foo = type('Foo', (), {'bar':True})
```
并作为一个普通的类使用：

```python
>>> print(Foo)
<class '__main__.Foo'>
>>> print(Foo.bar)
True
>>> f = Foo()
>>> print(f)
<__main__.Foo object at 0x8a9b84c>
>>> print(f.bar)
True
```
当然，你可以继承它，所以：

```python
>>>   class FooChild(Foo):
...         pass
```

将会：

```python
>>> FooChild = type('FooChild', (Foo,), {})
>>> print(FooChild)
<class '__main__.FooChild'>
>>> print(FooChild.bar) # bar is inherited from Foo
True
```

最终你会想要添加方法到你的类。只需定义一个具有适当签名的函数，并将其分配为一个属性即可。

```python
>>> def echo_bar(self):
...       print(self.bar)
...
>>> FooChild = type('FooChild', (Foo,), {'echo_bar': echo_bar})
>>> hasattr(Foo, 'echo_bar')
False
>>> hasattr(FooChild, 'echo_bar')
True
>>> my_foo = FooChild()
>>> my_foo.echo_bar()
True
```

在动态创建类之后，您可以添加更多的方法，就像向正常创建的类对象中添加方法一样。

```python
>>> def echo_bar_more(self):
...       print('yet another method')
...
>>> FooChild.echo_bar_more = echo_bar_more
>>> hasattr(FooChild, 'echo_bar_more')
True
```

您将看到我们要去的地方：在Python中，类是对象，您可以动态地创建一个类。

这就是Python在使用关键字class时所做的事情，而且它是通过使用元类来实现的。

什么是元类（最后）
元类是创建类的“东西”。

你定义类来创建对象，对吧？

但是我们知道Python类是对象。

那么，元类是创建这些对象。他们是类，你可以这样描绘他们：

MyClass = MetaClass()
MyObject = MyClass()
你已经看到，type让你做这样的事情：

MyClass = type('MyClass', (), {})
这是因为这个函数type实际上是一个元类。type是Python用来在幕后创建所有类的元类。

现在你想知道为什么它是用小写字母写的，而不是Type？

那么，我想这是一个与str创建字符串对象int的类以及创建整型对象的类保持一致的问题。type只是创建类对象的类。

你通过检查__class__属性来看到。

一切，我的意思是一切，都是Python中的一个对象。包括整数，字符串，函数和类。所有这些都是对象。所有这些都是从一个类创建的：

```python
>>> age = 35
>>> age.__class__
<type 'int'>
>>> name = 'bob'
>>> name.__class__
<type 'str'>
>>> def foo(): pass
>>> foo.__class__
<type 'function'>
>>> class Bar(object): pass
>>> b = Bar()
>>> b.__class__
<class '__main__.Bar'>
```

现在，这是__class__什么__class__？

```python
>>> age.__class__.__class__
<type 'type'>
>>> name.__class__.__class__
<type 'type'>
>>> foo.__class__.__class__
<type 'type'>
>>> b.__class__.__class__
<type 'type'>
```

所以，元类就是创建类对象的东西。

如果你愿意，你可以称之为“类工厂”。

type 是Python使用的内置元类，但是当然，您可以创建自己的元类。

该__metaclass__属性
__metaclass__当你写一个类时，你可以添加一个属性：

class Foo(object):
  __metaclass__ = something...
  [...]
如果这样做，Python将使用元类来创建类Foo。

小心，这很棘手。

你class Foo(object)先写，但是类对象Foo不是在内存中创建的。

Python将__metaclass__在类定义中寻找。如果找到它，它将使用它来创建对象类Foo。如果没有，它将  type用来创建类。

多读几遍。

当你这样做时：

class Foo(Bar):
  pass
Python执行以下操作：

有一个__metaclass__属性Foo？

如果是的话，在内存中创建一个类对象（我说的是类对象，留在我这里），名称Foo使用是什么__metaclass__。

如果Python找不到__metaclass__，它将__metaclass__在MODULE级别寻找一个，并尝试做同样的事情（但只适用于不继承任何东西的类，基本上是旧式的类）。

如果找不到任何东西__metaclass__，它将使用Bar自己的元类（可能是默认值type）来创建类对象。

在这里要小心，该__metaclass__属性不会被继承，父类（Bar.__class__）的元类将是。如果Bar使用的__metaclass__是创建的属性Bar与type()（不是type.__new__()），子类不会继承该行为。

现在最大的问题是，你能放__metaclass__什么？

答案是：可以创建一个类的东西。

什么可以创建一个类？type，或任何子类或使用它。

自定义元类
元类的主要目的是在创建时自动更改类。

您通常对API进行此操作，您需要创建与当前上下文相匹配的类。

想象一个愚蠢的例子，你决定你的模块中的所有类都应该用大写字母来写属性。有几种方法可以做到这一点，但一种方法是__metaclass__在模块级别设置。

这样，这个模块的所有类都将使用这个元类来创建，我们只需要告诉元类将所有的属性都改为大写。

幸运的是，`__metaclass__`实际上可以是任何可调用的，它不需要是一个正式的类（我知道，名称中的'class'不需要成为一个类，但是这是有用的）。

所以我们将从一个简单的例子开始，使用一个函数。

```python
# the metaclass will automatically get passed the same argument
# that you usually pass to `type`
def upper_attr(future_class_name, future_class_parents, future_class_attr):
  """
    Return a class object, with the list of its attribute turned
    into uppercase.
  """

  # pick up any attribute that doesn't start with '__' and uppercase it
  uppercase_attr = {}
  for name, val in future_class_attr.items():
      if not name.startswith('__'):
          uppercase_attr[name.upper()] = val
      else:
          uppercase_attr[name] = val

  # let `type` do the class creation
  return type(future_class_name, future_class_parents, uppercase_attr)

__metaclass__ = upper_attr # this will affect all classes in the module

class Foo(): # global __metaclass__ won't work with "object" though
  # but we can define __metaclass__ here instead to affect only this class
  # and this will work with "object" children
  bar = 'bip'

print(hasattr(Foo, 'bar'))
# Out: False
print(hasattr(Foo, 'BAR'))
# Out: True

f = Foo()
print(f.BAR)
# Out: 'bip'
现在，让我们做同样的事情，但使用一个真正的类为一个元类：

# remember that `type` is actually a class like `str` and `int`
# so you can inherit from it
class UpperAttrMetaclass(type):
    # __new__ is the method called before __init__
    # it's the method that creates the object and returns it
    # while __init__ just initializes the object passed as parameter
    # you rarely use __new__, except when you want to control how the object
    # is created.
    # here the created object is the class, and we want to customize it
    # so we override __new__
    # you can do some stuff in __init__ too if you wish
    # some advanced use involves overriding __call__ as well, but we won't
    # see this
    def __new__(upperattr_metaclass, future_class_name,
                future_class_parents, future_class_attr):

        uppercase_attr = {}
        for name, val in future_class_attr.items():
            if not name.startswith('__'):
                uppercase_attr[name.upper()] = val
            else:
                uppercase_attr[name] = val

        return type(future_class_name, future_class_parents, uppercase_attr)
```

但是这不是真的OOP。我们type直接调用，我们不会覆盖或调用父母__new__。我们开始做吧：

```python
class UpperAttrMetaclass(type):

    def __new__(upperattr_metaclass, future_class_name,
                future_class_parents, future_class_attr):

        uppercase_attr = {}
        for name, val in future_class_attr.items():
            if not name.startswith('__'):
                uppercase_attr[name.upper()] = val
            else:
                uppercase_attr[name] = val

        # reuse the type.__new__ method
        # this is basic OOP, nothing magic in there
        return type.__new__(upperattr_metaclass, future_class_name,
                            future_class_parents, uppercase_attr)
```

你可能已经注意到了额外的争论upperattr_metaclass。没有什么特别之处：`__new__`总是接收它定义的类作为第一个参数。就像你有self普通方法接收实例作为第一个参数，或类方法的定义类一样。

当然，为了清楚起见，我在这里使用的名字是很长的，但是self所有的论点都有传统的名字。所以一个真正的生产元类将如下所示：

```python
class UpperAttrMetaclass(type):

    def __new__(cls, clsname, bases, dct):

        uppercase_attr = {}
        for name, val in dct.items():
            if not name.startswith('__'):
                uppercase_attr[name.upper()] = val
            else:
                uppercase_attr[name] = val

        return type.__new__(cls, clsname, bases, uppercase_attr)
```

我们可以通过使用它来更简洁super，这将减轻继承（因为是的，你可以有元继承，继承自类继承的元类）：

```python
class UpperAttrMetaclass(type):

    def __new__(cls, clsname, bases, dct):

        uppercase_attr = {}
        for name, val in dct.items():
            if not name.startswith('__'):
                uppercase_attr[name.upper()] = val
            else:
                uppercase_attr[name] = val

        return super(UpperAttrMetaclass, cls).__new__(cls, clsname, bases, uppercase_attr)
```

而已。关于元类没有什么更多。

使用元类代码的复杂性背后的原因不是因为元类，这是因为你通常使用元类来依靠内省来做扭曲的东西，操纵继承，变量__dict__等等。

事实上，元类对于做黑魔法特别有用，因此也是复杂的东西。但是它们本身就很简单：

拦截一个类的创建
修改类
返回修改的类
为什么你会使用元类而不是函数？
既然__metaclass__可以接受任何可调用，为什么你会使用一个类，因为它显然更复杂？

这样做有几个原因：

意图很清楚。当你阅读时UpperAttrMetaclass(type)，你会知道接下来会发生什么
你可以使用OOP。元类可以从元类继承，重写父类的方法。元类甚至可以使用元类。
如果你指定了一个元类，而不是一个元类函数，那么类的子类将是它的元类的实例。
你可以更好地构建你的代码。你从来没有像上面的例子那样使用元类。这通常是复杂的。有能力创建几个方法并将它们分组在一个类中，这对于使代码更易于阅读非常有用。
你可以钩子`__new__`，`__init__`和`__call__`。这将允许你做不同的东西。即使通常你可以做到这一点__new__，有些人更舒适的使用__init__。
这些被称为元类，该死的！这意味着什么！
你为什么要使用元类？
现在是个大问题。为什么你会使用一些模糊的错误倾向功能？

那么，通常你不会：

元类是更深的魔法，99％的用户不应该担心。如果你想知道你是否需要他们，你不需要（那些真正需要他们的人肯定知道他们需要他们，不需要解释为什么）。

Python大师蒂姆·彼得斯

元类的主要用例是创建一个API。一个典型的例子是Django的ORM。

它允许你定义这样的东西：

```python
class Person(models.Model):
  name = models.CharField(max_length=30)
  age = models.IntegerField()
```
但是，如果你这样做：

```python
guy = Person(name='bob', age='35')
print(guy.age)
```
它不会返回一个IntegerField对象。它会返回一个int，甚至可以直接从数据库中获取。

这是可能的，因为models.Model定义__metaclass__和它使用了一些魔法，将Person您刚刚定义的简单语句变成一个复杂的钩子到数据库字段。

Django通过暴露一个简单的API并使用元类来创建一些复杂的外观，从这个API重新创建代码，在幕后做真正的工作。

最后一个字
首先，您知道类是可以创建实例的对象。

实际上，类本身就是实例。元类

```python
>>> class Foo(object): pass
>>> id(Foo)
142630324
```
一切都是Python中的对象，它们都是类的实例或元类的实例。

除了type。

type实际上是它自己的元类。这不是在纯Python中可以重现的东西，而是通过在实现级别上作弊。

其次，元类是复杂的。你可能不想用它们来进行非常简单的课堂改动。你可以通过使用两种不同的技术来改变类：

猴子补丁
类装饰器
99％的时间你需要改变类，你最好使用这些。

但98％的时间，你根本不需要改变类。
