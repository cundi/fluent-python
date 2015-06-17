# fluent-python
《精通Python》
### 英文原版：
http://shop.oreilly.com/product/0636920032519.do  

### 原书名：
《Fluent Python 副标题 Clear, Concise, and Effective Programming》  

### 特色： 
>Learn how to write idiomatic, effective Python code by leveraging its best features. Python's simplicity quickly lets you become productive with it, but this often means you aren’t using everything the language has to offer. By taking you through Python’s key language features and libraries, this practical book shows you how to make your code shorter, faster, and more readable all at the same time—what experts consider Pythonic.

>Many programmers who learn Python basics fall into the trap of reinventing the wheel because of past experience in other languages, and try to bend the language to patterns that don't really apply to it. Author Luciano Ramalho, a Python Software Foundation member and Python programmer for 15 years, helps you drop your accent from another language so you can code Python fluently.

>Learn practical applications of generators for database processing
Rethink some design patterns in a Python context
Examine attribute descriptors and when to use them: the key to ORMs
Explore Pythonic objects: protocols versus interfaces, abstract base classes and multiple inheritance
  
### 作者：
Luciano Ramalho  

### 出版社: 
O'Reilly Media  

## 纸书出版日期：
August 2015 (美国东部时间.)  

## 全书页数: 
750  

*************
  

`第十二章预览`  
Chapter 12. Inheritance: for good or for worse  
第十二章 继承该如何是好
**********************************************
  
>［我们］推动了继承思想，使其成为新手也可以构建框架的一种方法，而原先只有专家才可以设计的框架。

>— 阿兰。凯《Smalltalk的早期历史》
  
本章有关于继承和子类化，其中有两个针对Python不同的重点内容：  

* 从内建类型中的子类化陷阱  
* 多重继承以及方法解析顺序  

很多人认为多重继承带来的麻烦远大于其自身带来好处。  

然而，Java出奇的成功和顺利，这意味着，在实际操作中很多程序员并没有见到多重继承。这就是为什么我们通过两个重要的项目来阐明多重继承的适应范围：`Tkinter GUI`套件，以及Django web 框架。  

我们从内建子类化的问题开始。余下的章节会用案例研究来学习多重继承，并讨论在构建类的分层设计时所遇到的问题。  

## 技巧之-子类化内建类型
在Python2.2之前，子类化`list`或者`dict`这样的内建类型是不可能的。打那以后，Python虽然可以做到子类化内建类型，但是仍然要面对的重要警告是：内建的代码（由C语言重写）并不会调用被通过用户自定义类所覆盖的特殊方法。  

对问题的准确描述都放在了`PyPy`文档，以及内建类型的子类化一节中的`PyPy和CPython之间差异`：  

>正式地来说，Cpython对完全地重写内建类型的子类方法时是否明确地调用毫无规则可循。大略上，这些方法从来没有被其他的相同对象的内建方法所调用。例如，`dict`子类中的重写`__getitem__()`不会被`get()`这样的内建方法调用。  

例子12-1阐明了此问题。  

*例子12-1。重写的`__setitem__`被`dict`的`__init__`和`__update__`方法所忽略。*
  
************************
  
```python
>>> class DoppelDict(dict):
...     def __setitem__(self, key, value):
...         super(DoppelDict, self).__setitem__(key, [value] * 2)  # 1...
>>> dd = DoppelDict(one=1)  # 2
>>> dd
{'one': 1}
>>> dd['two'] = 2  # 3
>>> dd
{'one': 1, 'two': [2, 2]}
>>> dd.update(three=3)  # 4>
>> dd
{'three': 3, 'one': 1, 'two': [2, 2]}
```
  
1:存储时`DoppelDict.__setitem__`会使值重复（由于这个不好原因，因此必须有可见的效果）。它在委托到超类时才会正常运行。  

2:继承自`dict`的`__init__`方法，明确地忽略了重写的`__setitem__`：`'one'`的值并没有重复。  

3：`[]`运算符调用`__setitem__`，并如所希望的那样运行：`'two'`映射到了重复的值`[2， 2]`。  

4：`dict`的`update`方法也没有使用我们定义的`__setitem__`：值`'three'`没有被重复。  

该内建行为违反了面向对象的基本准则：方法的搜索应该总是从目标实例（`self`）的类开始，甚至是调用发生在以超类实现的方法之内部。在这样的悲观的情形下，

问题是在一个实例内部没有调用的限制，例如，不论`self.get()`是否调用`self.__getitem__()`，都会出现会被内建方法所调用其他类的方法被重写。下面是改编自`PyPy文档`的例子：  

例子12-2。`AnswerDict`的`__getitem__`被`dict.update`所忽略。  

```python
>>> class AnswerDict(dict):
...     def __getitem__(self, key):  # 1...
return 42
...
>>> ad = AnswerDict(a='foo')  # 2
>>> ad['a']  # 3
42
>>> d = {}
>>> d.update(ad)  # 4
>>> d['a']  # 5
'foo'
>>> d
{'a': 'foo'}
```
  
1：`AnserDict.__getitem__`总是返回`42`,不论键是什么。  

2：`ad`是一个带有键值对`('a', 'foo')`的`AnswerDict`。  

3：`ad['a']`如所期望的那样返回42。  

4：`d`是一个普通使用`ad`更新的`dict`实例。  

5：`dict.update`方法忽略了`AnserDict.__getitem__`。  

>##### 警告
直接地子类化类似`dict`或者`list`或者`str`这样的内建类型非常容易出错，因为大多数的内建方法会忽略用户所定义的重写方法。从被设计成易于扩展的`collections`模块的`UserDict`，`UserList`和`UserString`派生类，而不是子类化内建。  

如果你子类化`collections.UserDict`而不是`dict`，那么例子12-1和例子12-2中的问题都会被该解决。见例子12-3。  

*例子12-3。`DoppelDict2`和`AnswerDict2`一如所希望的运行，因为它们扩展的是UserDict而不是dict。*
***************
  
```python
>>> import collections
>>>
>>> class DoppelDict2(collections.UserDict):
...     def __setitem__(self, key, value):
...         super().__setitem__(key, [value] * 2)
...
>>> dd = DoppelDict2(one=1)
>>> dd
{'one': [1, 1]}
>>> dd['two'] = 2
>>> dd
{'two': [2, 2], 'one': [1, 1]}
>>> dd.update(three=3)
>>> dd
{'two': [2, 2], 'three': [3, 3], 'one': [1, 1]}
>>>
>>> class AnswerDict2(collections.UserDict):
...     def __getitem__(self, key):
...         return 42
...
>>> ad = AnswerDict2(a='foo')
>>> ad['a']
42
>>> d = {}
>>> d.update(ad)
>>> d['a']
42
>>> d
{'a': 42}
```
  
为了估量内建的子类工作所要求体验，我重写了例子3-8中`StrKeyDict`类。继承自`collections.UserDict`的原始版本，由三种方法实现：`__missing__`，`___contains__`和`__setitem__`。

总结：本节所描述的问题仅应用于在C语言内的方法委托实现内建类型，而且仅对用户定义的派生自这些的类型的类有效果。如果你在Python中子类化类编程，比如，`UserDict`或者`MutableMapping`，你不会遇到麻烦的。  

还有问题就是，有关继承，特别地的多重继承：Python如何确定哪一个属性应该使用，如果超类来自并行分支定义相同的名称的属性，答案在下面一节。  

### 多重继承以及方法解析顺序
当不关联的祖先类实现相同名称的方法时，任何语言实现多重继承都需要解决潜在的命名冲突。这称做“钻石问题”，一如图表12-1和例子12-4所描述。  

图片：略
  

图表12-1.左边：UML类图表阐明了“钻石问题”。右边：虚线箭头为例子12-4描绘了Python MRO（方法解析顺序）.  
例子12-4. diamond.py：类A，B， C，和D构成了图表12-1中的图。  

```python
class A:
    def ping(self):
        print('ping:', self)


class B(A):
    def pong(self):
        print('pong:', self)


class C(A):
    def pong(self):
        print('PONG:', self)


class D(B, C):

    def ping(self):
        super().ping()
        print('post-ping:', self)

    def pingpong(self):
        self.ping()
        super().ping()
        self.pong()
        super().pong()
        C.pong(self)

```
  
注意类`B`和`C`都实现了`pong`方法。唯一的不同是`C.pong`输出大写的单词`PONG`。  

如果你对实例`D`调用`d.pong()`，实际上哪一个`pong`方法会运行呢？对于C++程序员来说他们必须具有使用类名称调用方法，以解决这个模棱两可的问题。这样的问题在Python中也能够解决。看下例子12-5就知道了。  

例子12-5.对类D的实例的pong方法调用的两种形式。  

```python
>>> from diamond import *
>>> d = D()
>>> d.pong()  #  1
pong: <diamond.D object at 0x10066c278>
>>> C.pong(d)  #  2
PONG: <diamond.D object at 0x10066c278>
```
1: 简单地调用`d.pong`导致B的运行。  
2: 你可以总是直接地对调用超类的方法，传递实例作为明确的参数。  

像`d.pong()`这样的模棱两可的调用得以解决，因为Python在穿越继承图时，遵循一个特定的顺序。这个顺序就叫做MRO：方法解析顺序。类有一个被称为`__mro__`的属性，它拥有使用MRO顺序的超类的引用元组，即，当前的类的所有到`object`类的路径。拿类`D`来说明什么是`__mro__`（参见 图表12-1）：  

```python
>>> D.__mro__
(<class 'diamond.D'>, <class 'diamond.B'>, <class 'diamond.C'>,
<class 'diamond.A'>, <class 'object'>)
```
  
推荐的调用超类的委托方法就是内建的`super()`函数，这样做是因为在Python3中较易使用，就像例子12-4中的类D的`pingpong`方法所阐述的那样。不过，有时候忽略MRO，对超类直接地调用方法也是也可以的，而且很方便。例如，`D.ping`方法可以这样写：  

```python
    def ping(self):
        A.ping(self)  # instead of super().ping()
        print('post-ping:', self)
```
  
注意，当调用直接调用一个类的实例时，你必须明确地传递`self`，因为你访问的是`unbound method`。  

不过，这是最安全的而且更未来化的使用`super()`，特别是在调用一个框架的方法时，或者任何不受你控制的类继承时。例子12-6演示了在调用方法时`super()`对MRO的遵循。  

例子12-6。使用`super()`去调用`ping`（源码见例子12-4）。  

```python
```

  
## 真实世界中的多重继承
## 应对多重继承
#### 1. 接口继承和接口实现之间的区别

##### 继承

#### 2. 使用ABC让接口更清晰
#### 3. 为了代码重复利用而使用mixin

## 一个当代的例子：Django通用视图中的mixin
>##### 注释
