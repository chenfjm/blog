---
layout: post
title: "Python参数传递"
description: "关于python的参数传递"
category: Python
---

参数传递方式
==========

 * 确定个数的参数
 * 确定个数的关键字参数
 * 不定个数的参数
 * 不定个数的关键字参数
 
确定个数的参数
------------

这是最常见的参数传递方式，大部分的函数参数传递方式都是这种方式。它要求实参和虚参的个数相同，顺序也要相同，也就是要保持一一对应，否则就会出现错误。

    def foo(num):
        print num
        
    if __name__ == "__main__":
        number = 10
        foo(number)
        
确定个数的关键字参数
-----------------
这种方式传递参数需要指定参数的名字。

    def foo(Name='Tom', Age=20, Tall＝None):
        print 'Name:', Name
        print 'Age:', Age
        print 'Tall:', Tall
    
    if __name__ == "__main__":
        name = 'lucy'
        age = 19
        tall = 168
        foo(Name=name, Age=age, Tall=tall)
        
使用这种方式进行参数传递的时候，需要指定参数的名字，以键值对的方式进行传递，如果某个键缺省，则使用默认值。这种方式的传递在Python中使用比较普遍。

不定个数的参数
------------

有时候我们设计一个函数的时候，并不知道它接受的参数的个数。只是知道需要对传递进来的参数依次进行处理。这种情况下就需要使用不定参数传递方式。在Python中传递这种不定参数通过在参数名前面放一个*来实现。如*args

    def foo(*args):
        for arg in args:
            print arg
            
    if __name__ == "__main__":
        name = "Jim"
        age = 22
        foo(name, age)
        tall = 178
        foo(name, age, tall)
        
不定个数的关键字参数
-----------------

不定个数的关键字参数传递和上面的参数传递方式有些相似，这里通过在参数名字前面添加**来实现，如**kwargs

    def foo(**kwargs):
        for k, v in kwargs.items():
            print k, v
            
    if __name__ == "__main__":
        name = 'Jin'
        age = 22
        foo(Name=name, Age=age)
        tall = 178
        foo(Name=name, Age=age, Tall=tall)
        
在这里可以发现，对于关键字的名字我们也是可以任意指定的，函数调用的内部会去获取关键字的名字和值。

特殊的参数传递
============

未知个数的变量列表作为参数
----------------------

在编程过程中，我不止依次遇到过这样的情况：在现有的程序运行下，我得到了一个列表，也就是一系列的变量，对于这些变量，我并不知道它们的具体个数，也不知道它们的值，它们只是保存在一个列表或者是集合中。而现在我需要把它传递给一个不定参数的函数，让这个函数来处理这些列表里面的变量。描述成代码，情况如下：

    def foo(*args):
        do something
        
    def bar():
        return a list or a tuple
        
    if __name__ == "__main__":
        var = bar()
        # 我要把var中的每一个元素作为一个参数传递给foo函数
        # 如果var内容是 [var1, var2, var3]
        # 那么我需要调用foo(var1, var2, var3)
        # 显然这里调用foo(var)是错误的
        # 于是需要使用以下方式来实现这样的功能
        foo(*tuple(var)) ＃这里的*var用于unpack一个元组
        
这种方式的传递参数，我曾在使用lua编程的时候遇到过，在lua中允许函数返回多个返回值，并且可以通过loadsrting执行一段字符串代码。所以我可以通过编写一个函数来解决。在Python中则自带了这样的方法。

未知个数和名字的关键字参数字典
-------------------------

和上面传递参数情形一样，这是这里需要传递的是一个key,value对。而上面的列表在这里编程了一个未知个数，未知key的字典。使用方法如下:

    def foo(**kwargs):
        do something
        
    def bar():
        return a dict
        
    if __name__ == "__main__":
        var = bar()
        foo(**var) #通过在前面加两个*号来unpack一个字典
        
这种参数使用方式在Tornado中就会遇到，在给tornado.web.Application类传递参数的时候，往往会传递一个**settings的参数，settings是我们在外面定义好的一个字典。

总结
===

Python中的参数传递主要为我第一部分说的四种，但是在某些编写环境下，会用到第二部分讲到的比较特殊的参数传递方式。至少我自己在Python的编程过程中都遇到过这样的情景。
        
 



