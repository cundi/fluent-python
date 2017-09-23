# CHAPTER 15 Context Managers and else Blocks

    Context managers may end up being almost as important as the subroutine itself. We’ve only scratched the surface with them. [...] Basic has a with statement, there are with statements in lots of languages. But they don’t do the same thing, they all do something very shallow, they save you from repeated dotted [attribute] lookups, they don’t do setup and tear down. Just because it’s the same name don’t think it’s the same thing. The with statement is a very big deal. [1]
    — Raymond Hettinger Eloquent Python evangelist

[#1]PyCon US 2013 keynote: “What Makes Python Awesome”; the part about with starts at 23:00 and ends at 26:15.  

In this chapter, we will discuss control flow features that are not so common in other languages, and for this reason tend to be overlooked or underused in Python. They are:  

本章，我们会讨论在其他语言中并不常见的`控制流`功能，

• The with statement and context managers
• The else clause in for, while, and try statements

- with语句和上下文管理器
- for、while和try语句中的else子句

The with statement sets up a temporary context and reliably tears it down, under the control of a context manager object. This prevents errors and reduces boilerplate code, making APIs at the same time safer and easier to use. Python programmers are finding lots of uses for with blocks beyond automatic file closing.  

with语句建立一个临时的上下文，并在上下文管理器对象的控制下可靠的销毁它。

The else clause is completely unrelated to with. But this is Part V, and I couldn’t find another place for covering else, and I wouldn’t have a one-page chapter about it, so here it is.  

else子句和with完全没有联系。但是

Let’s review the smaller topic to get to the real substance of this chapter.  


