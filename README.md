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
在Python2.2之前，子类化`list`或者`dict`这样的内建类型是不可能的。自打那以后，Python虽然可以做到子类化内建类型，但是仍然要面对的重要警告是：内建的代码（由C语言重写）并不会调用被通过用户自定义类所覆盖的特殊方法。  

对问题的准确描述都放在了`PyPy`文档，以及内建类型的子类化一节中的`PyPy和CPython之间差异`：  

>正式地来说，Cpython对完全地重写内建类型的子类方法时是否明确地调用毫无规则可循。大略上，这些方法从来没有被其他的相同对象的内建方法所调用。例如，`dict`子类中的重写`__getitem__()`不会被`get()`这样的内建方法调用。  

例子12-1阐明了此问题。  

*例子12-1。重写的`__setitem__`被`dict`的`__init__`和`__update__`方法所忽略。*
  
************************
  
```python
>>> class DoppelDict(dict):
...     def __setitem__(self, key, value):
...         super().__setitem__(key, [value] * 2)  # 1...
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
  
1:存储时`DoppelDict.__setitem__`会使值重复（由于这个原因，因此必须有可见的效果）。它在派遣到超类时才会正常运行。  

2:`__init__`方法继承自`dict`，明确地忽略了重写的`__setitem__`：``one``的值并没有重复。  

3：`[]`运算符调用`__setitem__`，并如所希望的那样运行：``two``映射重复的值`[2， 2]`。  

4：`dict`的`update`方法也没有使用我们定义的`__setitem__`：值``three``没有被重复。  
