Chapter 1. The Python Data Model
********************************
  
*Guido’s sense of the aesthetics of language design is amazing. I’ve met many fine language designers who could build theoretically beautiful languages that no one would ever use, but Guido is one of those rare people who can build a language that is just slightly less theoretically beautiful but thereby is a joy to write programs in[3].*

*— Jim Hugunin creator of Jython, co-creator of AspectJ, architect of the .Net DLR*  

One of the best qualities of Python is its consistency. After working with Python for a while, you are able to start making informed, correct guesses about features that are new to you.  


However, if you learned another object oriented language before Python, you may have found it strange to spell `len(collection)` instead of `collection.len()`. This apparent oddity is the tip of an iceberg which, when properly understood, is the key to everything we call `Pythonic`. The iceberg is called the Python Data Model, and it describes the API that you can use to make your own objects play well with the most idiomatic language features.  

You can think of the Data Model as a description of Python as a framework. It formalizes the interfaces of the building blocks of the language itself, such as sequences, iterators, functions, classes, context managers and so on.  

While coding with any framework, you spend a lot of time implementing methods that are called by the framework. The same happens when you leverage the Python Data Model. The Python interpreter invokes special methods to perform basic object operations, often triggered by special syntax. The special method names are always spelled with leading and trailing double underscores, i.e. `__getitem__`. For example, the syntax `obj[key]` is supported by the `__getitem__` special method. To evaluate `my_collection[key]`, the interpreter calls `my_collection.__getitem__(key)`.  

The special method names allow your objects to implement, support and interact with basic language constructs such as:  

    iteration;
    collections;
    attribute access;
    operator overloading;
    function and method invocation;
    object creation and destruction;
    string representation and formatting;
    managed contexts (i.e. *with* blocks);

#####MAGIC AND DUNDER
The term magic method is slang for special method, but when talking about a specific method like `__getitem__`, some Python developers take the shortcut of saying “under-under-getitem” which is ambiguous, since the syntax __x has another special meaning[4]. But being precise and pronouncing “under-under-getitem-under-under” is tiresome, so I follow the lead of author and teacher Steve Holden and say “dunder-getitem”. All experienced Pythonistas understand that shortcut. As a result, the special methods are also known as dunder methods [5].  

##A Pythonic Card Deck
The following is a very simple example, but it demonstrates the power of implementing just two special methods, `__getitem__` and `__len__`.  

Example 1-1 is a class to represent a deck of playing cards:  

*Example 1-1. A deck as a sequence of cards.*  

```python
import collections

Card = collections.namedtuple('Card', ['rank', 'suit'])

class FrenchDeck:
    ranks = [str(n) for n in range(2, 11)] + list('JQKA')
    suits = 'spades diamonds clubs hearts'.split()

    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits
                                        for rank in self.ranks]

    def __len__(self):
        return len(self._cards)

    def __getitem__(self, position):
        return self._cards[position]
```
