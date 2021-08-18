---
layout: post
title:  "Python Special Variables in Classes"
date:   2021-01-01 01:01:00 +0000
categories: jekyll update
---
Below are some attributes defined by python (by default) in all Classes:

* classname.__bases__ : Tuple of base classes a class inherits from. This attribute only returns the classes one level up in the inheritance hierarchy.
* classname.__name__: Name of the class
* classname.__module__: The module name in which the class was defined.
* classname.__dict__ : Dictionary containing the class's namespace.

Following is a sample program that prints these default attributes. 

```
$ cat print_class_attrs.py
import logging

class A():
    classAattr = "Attr in Class A"
    pass

class B():
    classBattr = "Attr in Class B"
    pass

class C(A,B):
    classCattr = "Attr in Class C"
    pass

class D(C):
    classDattr = "Attr in Class D"
    pass

if __name__ == '__main__':

    print "Class C's Attributes:"
    print "bases :", C.__bases__
    print "name :", C.__name__
    print "module name :", C.__module__
    print "C's internal dict :", C.__dict__
    # the builtin function dir() can also be used to get a list attributes in a class.
    print "Attributes in Class C from dir", dir(C)

    print
    print
    # Only prints classes one level up in the inheritence hierarchy. This doesn't print classes A,B in the hierarchy.
    print "Class D's Attributes:"
    print D.__bases__
    print "D's internal dict :",D.__dict__
    print
    print
    print "module name of Formatter class in logging module :", logging.Formatter.__module__
```

**Output**

```

Class C's Attributes:
bases : (<class __main__.A at 0x7fe5886509a8>, <class __main__.B at 0x7fe588650a10>)
name : C
module name : __main__
C's internal dict : {'classCattr': 'Attr in Class C', '__module__': '__main__', '__doc__': None}
Attributes in Class C from dir ['__doc__', '__module__', 'classAattr', 'classBattr', 'classCattr']


Class D's Attributes:
(<class __main__.C at 0x7fe588650a78>,)
D's internal dict : {'__module__': '__main__', '__doc__': None, 'classDattr': 'Attr in Class D'}


module name of Formatter class in logging module : logging
```
