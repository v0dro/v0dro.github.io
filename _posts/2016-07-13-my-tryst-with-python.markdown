---
layout: post
title: "My tryst with Python"
date: 2016-07-13 16:24:23 +0530
comments: true
published: false
categories: 
---

A particular course in college called Computational Problem Solving required me to learn Python and use it as a demo language for all sorts of computer science problems involving sorting, searching, types of algorithms and different types of data structures. I'm a Rubyist at heart and not at all a fan of Python and will not use the language unless I have to. This rather lengthy blog post is for documenting whatever I did with Python for this particular course.

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Sorting](#sorting)
    - [-](#-)
    - [Insertion sort](#insertion-sort)
    - [Selection Sort](#selection-sort)
    - [Quick sort](#quick-sort)
    - [Heap sort](#heap-sort)
- [Printing directory contents](#printing-directory-contents)
    - [Lessons learnt](#lessons-learnt)
- [Strings](#strings)
- [Zipping in Python](#zipping-in-python)
- [Weird python keywords](#weird-python-keywords)
    - [nonlocal](#nonlocal)
    - [in keyword](#in-keyword)
        - [Inside if statements](#inside-if-statements)
        - [Inside for statements](#inside-for-statements)
- [Mutable (saved) function arguments](#mutable-saved-function-arguments)
- [Decorators](#decorators)
- [The super method](#the-super-method)
- [Special methods](#special-methods)
    - [Equality of objects](#equality-of-objects)
    - [Getting items](#getting-items)
- [The Garbage Collector](#the-garbage-collector)
- [CLI programs](#cli-programs)
- [Packaging and creating your own modules](#packaging-and-creating-your-own-modules)
- [C extensions](#c-extensions)
    - [Module initialization](#module-initialization)
    - [Method calling](#method-calling)
    - [Instance methods](#instance-methods)
        - [Magic methods](#magic-methods)
        - [PyMappingMethods](#pymappingmethods)
    - [Class methods](#class-methods)
    - [Error handling](#error-handling)
    - [Specifying class properties](#specifying-class-properties)
    - [PyCapsule a.k.a providing C API for other extension modules](#pycapsule-aka-providing-c-api-for-other-extension-modules)
    - [Creating Python objects from C structs](#creating-python-objects-from-c-structs)

<!-- markdown-toc end -->

# Sorting

I have implemented quite a few sorting algorithms in Python. I will document each of them 
and my interpretation of these and also post sample code.

#### Bubble sort

#### Insertion sort

Insertion basically maintains two lists within a single unsorted list: the partially sorted list and the unsorted the list. When sorting in ascending order the sorted list is to the left side.

This algorithm will start with the first element and keep it unchanged. It will then look at the second element, and if it is smaller than the first element it will shift the first element to second position and insert the second element in the position where the first element previously was. It proceeds in a similar fashion for the entire list. First see the element immediately to the right of the sorted list, if it is greater than the last element of the sorted list (i.e the element right before it) then let it be the way it is, otherwise scan the sorted list, see where the element can fit in, make space for that element and shift the list forward at that position such that space is made for that element and it can be inserted there.

In pseudo code:
```
while all_elements_have_not_been_checked
  if element_preceding_current_element > current_element
    pick_up_current_element
    scan_sorted_list_for_number_smaller_than_current_element
    shift_sorted_list_forward_by_one_position
    insert_number_in_now_vacant_position
```

This GIF from [Wikipedia](https://en.wikipedia.org/wiki/Insertion_sort) explains it pretty well:
![Insertion sort](/assets/images/tryst_with_python/insertion_sort.gif)

Here's my python script. Python experts are welcome to suggest edits to make it faster/smaller.
``` python
# Insertion sort in Python

import random

arr = []
for x in xrange(0,1000):
  arr.append(random.randint(1,1000))

def scan_arr_for_correct_position(i, arr):
  current = arr[i]
  for x in xrange(0,i):
    if current < arr[x]:
      return x

i = 1
while i < len(arr):
  if arr[i-1] > arr[i]:
    current = arr[i]
    pos = scan_arr_for_correct_position(i, arr)
    for x in reversed(xrange(pos,i+1)):
      arr[x] = arr[x-1]

    arr[pos] = current

  i += 1

print(arr)
```

Worst case time complexity of this algorithm is O(n^2). The best case performance is O(n). 
This algorithm is better than selection sort since it is adapative and does not necessarily 
need to swap elements if they are already in sorted order.

#### Selection Sort

Worst case time complexity of this algorithm is O(n^2). It differs from insertion sort
in a way that insertion sort picks up the first element after the sorted sublist (in the 
unsorted sublist) and finds a place for it in the sorted sublist, while selection sort 
selects the smallest element in the unsorted sublist and adds it to the end of the sorted sublist.

#### Quick sort

This has worst case time complexity of O(n^2) if swapping needs to be done for every element, 
but this behaviour is rare. Average case time complexity is O(nlog(n)).

#### Heap sort

This is similar to insertion sort, but the difference is that a heap data structure is used for getting the largest element from the unsorted list. For this reason, it has a worst case complexity of O(nlog(n)). Best case time complexity is O(n) or O(nlog(n)).

# Printing directory contents

A sample problem given is this:
``` python
def print_directory_contents(sPath):
    """
    This function takes the name of a directory 
    and prints out the paths files within that 
    directory as well as any files contained in 
    contained directories. 

    This function is similar to os.walk. Please don't
    use os.walk in your answer. We are interested in your 
    ability to work with nested structures. 
    """
   pass
```

I wrote the following function to demonstrate my usage of nested structures in Python.
```
import os
from os import listdir

def really_get_contents(s, indent, path):
    contents = listdir(path)
    
    for content in contents:
        s += "-"*indent + str(content) + "\n"
        if os.path.isdir(path + content):
            s = really_get_contents(s, indent+2, path + content + "/")
            
    return s

"""
Print the directory contents recursively of every directory specified in path.
"""
def print_directory_contents(path):
    s = ""
    indent = 0
    return really_get_contents(s, indent, path)

s = print_directory_contents("/home/1/17M38101/gitrepos/hpc_lecture/")
print(s)
```

## Lessons learnt

* Python strings are immutable.
* Use `os.path.join` for joining two strings that represent paths. This makes it cross-platform.

# Strings

Strings are immutable in python in all cases. You can't even duplicate a string without taking
extreme measures. If you want to perform string operations like swapping characters you need
to store the chars in a list and swap things in the list instead.

Link:
* https://stackoverflow.com/questions/4605439/what-is-the-simplest-way-to-swap-char-in-a-string-with-python
* https://asoldatenko.com/can-i-copy-string-in-python-and-how.html

# Zipping in Python

For zipping together two arrays in Ruby, one can simply call `[1,2,3].zip ["a", "b", "c"]` and
it will return an Array like `[[1, "a"], [2, "b"], [3, "c"]]`.

However in Python, the built-in `zip` function returns an iterable object using which you
can iterate over the zipped values. For example:
```
In [6]: zip([1,2,3], ["a", "b", "c"])
Out[6]: <zip at 0x7f582adc1888>
```
The iterator contains pairs of tuples. You can then create a list out of these tuples using
`list(zip([1,2,3], ["a", "b", "c"]))`, or even a `dict` if you use the `dict()` function.

# Weird python keywords

## nonlocal

Allows a closure to access and modify variables outside of its immediate scope.

Link: https://stackoverflow.com/questions/1261875/python-nonlocal-statement

## in keyword

### Inside if statements

This keyword is usually used in `if` statements to check if some elements exists in a list:
``` python
a = [1,2,3]
if 1 in a:
    print("yes!")
```

However, when it is used with a dict like so:
``` python
a = {1 : "a", 2 : "b", 3 : "c"}
if "a" in a:
    print("yes!")
else:
    print("no!")
```
It checks whether a particular key is present in the dict or not.

Link: https://pycruft.wordpress.com/2010/06/10/pythons-in-keyword/

### Inside for statements

Used for iterating over the elements of a list or keys of a dict.

# Mutable (saved) function arguments

Consider the following code:
``` python
def f(x,l=[]):
    for i in range(x):
        l.append(i*i)
    print(l) 

f(2)
f(3,[3,2,1])
f(3)
```
The output of the third line is `[0, 1, 0, 1, 4]`(!!!!!).

This is because when the subsequent function call that uses the default argument is 
called, it uses the same memory block as the previous call. This is weird because
a function is supposed to a self-contained unit that is not affected by code outside
its scope.

List default arguments are better used by specifying `None` as the default and then
checking if the argument is actually `None` as assigning it to a `list` if yes.

Link: http://docs.python-guide.org/en/latest/writing/gotchas/

# Decorators

A decorator is a special kind of function that either takes a function and returns a
function, or takes a class and returns a class. The `@` behind it is just syntactic
sugar that allows you to decorate something in a way that's easy to read.

The idea is based on the fact that functions are first-class objects in Python (unlike
Ruby). Thus we can return functions or assign them variables like any other value. This
is the propery that allows defining functions inside other functions. Compared to Ruby,
this property is like passing a block to the method by defining a function and passing
the function instead of a closure.

A decorator allows you to create a function call that calls the decorated function with
the name that is specified in the decorator, so that you can call the function by its name
rather than passing it into a call to some other function. So for example:
``` python
@time_this
def func_a(stuff):
    do_important_thing()
```
...is exactly equal to:
``` python
def func_a(stuff):
    do_important_thing()
func_a = time_this(func_a)
```

It is also possible to pass arguments to decorators depending on what context you
want a particular function to be called. So you can define functions inside decorator
functions that get called based on some argument that you pass to the decorator when
defining it above a method/class. For example:
``` python
@requires_permission('administrator')
def delete_user(iUserId):
   """
   delete the user with the given Id. 
   This function is only accessible to users with administrator permissions
   """
```
An example of implementing such 'nested decorators' can be the following code:
``` python
def outer_decorator(*outer_args,**outer_kwargs):
    def decorator(fn):
        def decorated(*args,**kwargs):
            do_something(*outer_args,**outer_kwargs)
            return fn(*args,**kwargs)
        return decorated
    return decorator
    
@outer_decorator(1,2,3)
def foo(a,b,c):
    print a
    print b
    print c

foo()
```
You can imagine the `outer_decorator` as being 'created' during the `@` call and the
`decorator` being placed in its place with the arguments `1,2,3` saved in the function
call. So now you can call the `decorator` decorator with whatever arguments you want
placed above the function call.

My personal take on decorators is that they feel a little jugaadu (Hindi for hack-y) and
can lead to problems if you don't read the decorator above a function and it does something
unexpected after you call it.

Link: 
* https://www.codementor.io/sheena/advanced-use-python-decorators-class-function-du107nxsv
* https://www.codementor.io/sheena/introduction-to-decorators-du107vo5c

# The super method

Unlike Ruby, the `super` keyword in Python returns a proxy object to delegate method calls
to a class. Its not just a method that calls the method of the same name in the super class.
Also, since Python supports multiple inheritance (ability for a single class to inherit from
multiple classes), this functionality allows users to specify the class from which they want
to call a particular method.

Link: http://www.pythonforbeginners.com/super/working-python-super-function

# Special methods

## Equality of objects

Over-ride the `__eq__` method.

## Getting items

The `[]` method can be used by over-riding the `__getitem__` method.

# The Garbage Collector

The Python interpreter maintains a count of references to each object in memory.
If a reference count goes to zero then the associated object is no longer alive 
and the memory allocated to that object can be freed. This is a different mechanism
from the Ruby GC, which scans the stack space for unused objects.

CPython uses a generational garbage collector alongwith the reference counting. This
is due to the presence of reference cycles. If an object contains references to other
objects, then their reference count is decremented too. Thus other objects may be
deallocated in turn.

Variables, which are defined inside blocks (e.g., in a function or class) have a 
local scope (i.e., they are local to its block). If Python interpreter exits from 
the block, it destroys all references created inside the block. The reference counting 
algorithm has a lot of issues, such as circular references, thread locking and memory 
and performance overhead.

The generational GC classifies objects into three generations. Every new object starts 
in the first generation. If an object survives a garbage collection round, it moves 
to the older (higher) generation. Lower generations are collected more often than 
higher. Because most of the newly created objects die young, it improves GC performance
and reduces the GC pause time.

I think this mechanism is both good and bad for C extension writers. Good because
you can explicitly maintain control on which objects get freed and which don't (using
references). Bad because it increases the complexity of C extensions (but there's 
Cython for that).

Link: https://rushter.com/blog/python-garbage-collector/

# CLI programs

CLI arguments are parsed using the `argparse` module.

In order to write a simple parser that accepts two arguments, `--major` and `--file`,
you can use code that looks like this:
``` python
parser = argparse.ArgumentParser(description="Remake matrix output by MPI processes.")
parser.add_argument('--file', dest="file", help="file extension name to read.")
parser.add_argument('--major', dest="major", default="col",\
    help="row or col major (default row)")
args = parser.parse_args()
```

It's a fairly straightforward module that can be used without much hassle.

Link:

* https://docs.python.org/3.3/howto/argparse.html#id1

# Packaging and creating your own modules

Follow the common directory structure, with the project root containing a folder
that is same as the name of the library name you're making. For example, if your
library is called 'sabrina':
```
```

## Python eggs

[Python eggs](https://setuptools.readthedocs.io/en/latest/formats.html) are strucutures that embody
the structure of a Python project, for example its code, metadata and other resources.

## The anaconda package manager

Anaconda provides a great package manager for Python with many useful features like setting up
environments, using different sources, etc. By default it will install into a folder called
`anaconda3` in your home directory. The major structure of this directory is as follows:
```
anaconda3/
  ├── bin
  ├── envs
  ├── include
  ├── lib
  ├── pkgs
  ├── var
```

The `pkgs` directory is an important [directory](https://stackoverflow.com/questions/51736615/explain-the-anaconda-folder-structure)
that installs the default packages that the distribution should contain when you
first initialize conda. All installed packages for the base environment are first
stored here using their canonical names.

When you derive a new environment, packages that you want to include in the new
env are included in the new environment by linking to packages kept in the `pkgs`
folder.

# C extensions

Python C extensions tend to be somewhat more complex than Ruby extensions because
you need to manually keep a count of the objects due to Python's use of a generational
GC (no mark-and-sweep) with reference counting. The exception handling mechanism is
also somewhat different from Ruby and is worth noting.

Python also uses a significantly different mechanism for calling C functions from
Python methods. Before being able to call functions, you need to 'register' the
function in a 'method table'.

Read on for details.

## Module initialization

Modules are initialized using the `PyInit_modulename` function. This function never
accepts any arguments and must be defined with argument `void`.

## Method calling

We need to first list the names and addresses of functions in a "method table". This
is how it can be done:
```
static PyMethodDef SpamMethods[] = {
    ...
    {"system",  spam_system, METH_VARARGS,
     "Execute a shell command."},
    ...
    {NULL, NULL, 0, NULL}        /* Sentinel */
};
```
Each entry in the table is an initialization for the struct of type `PyMethodDef`.
The 3rd argument specifies what kind of a Python method this will be and what
parameters will be passed to it when its gets called by the Python interpreter. Those
are flags that can OR'd. In the above case `METH_VARARGS` will create an instance
method.

This table should be passed to the initialization function of the module whenever
it gets called.

There are some special functions like `PyArg_ParseTupleAndKeywords` that can be
used for parsing arguments passed into a Python method.

Links:

* https://docs.python.org/2.0/ext/parseTupleAndKeywords.html

## Instance methods

Defining a C functions that is callable from Python requires you first write a function
with a prototype as:
```
static PyObject * method_name(PyObject *self, PyObject *args) {}
```
In the above example, the first arg `self` is a pointer to the Python instance that calls
this method (Python class) and `*args` is an array of Python objects that contains the
arguments.

### Magic methods

Magic methods like `__getitem__` need to be implmented separately using some different keywords.
The `sq_item` parameter needs to be a particular C function for it to function as the `[]`
operator on an object. You need to implment functions for the PySequence protocol.

Links:

* https://docs.python.org/3/c-api/typeobj.html#c.PySequenceMethods.sq_item
* https://docs.python.org/3/c-api/object.html#c.PyObject_GetItem

### PyMappingMethods

A lot of the information about a type is stored in a type called PyTypeObject. This struct
contains a field called `tp_mapping` that stores function pointers for various functions that
implement the mapping protocol for Python.

It basically helps to implement various 'array-like' functions like `len()` and the `[]` operator
on objects.

In order to implement the `[]` operator, you need to implement the function `mp_subscript` 
that will be called by the `PyObject_GetItem()` function that implements the `__getitem__`
magic function. This is like implementing the `[]` function in Ruby. The function that you
implement must have the same prototype as `PyObject_GetItem()`:
```
PyObject* PyObject_GetItem(PyObject *o, PyObject *key)
```
In the above function, `o` is the calling object (like `self`) and `key` is what is passed
into `[]`. So it would read like `o[key]`.

Links:

* https://docs.python.org/3/c-api/typeobj.html
* https://docs.python.org/3/c-api/typeobj.html#mapping-structs

## Class methods

If you want to mark a C function as a Python class method rather than an instance method,
you should pass the `METH_CLASS` parameter in the 3rd arg of the `PyMethodDef` initialization.

For example:
```
{ "deserialize", (PyCFunction)ndtype_deserialize, METH_O|METH_CLASS, doc_deserialize }
```

This will pass `PyTypeObject *` as the first parameter of the function instead of an
instance of the class. So the `ndtype_deserialize` function will look like so:
```
static PyObject * ndtype_deserialize(PyTypeObject *tp, PyObject *bytes);
```

Link:

* https://docs.python.org/3/c-api/structures.html#METH_CLASS

## Error handling

Python stores exceptions inside a static global variable. If no exceptions occur this
variable is `NULL`. If an exception occurs, the function is supposed to store the
exception handle in this variable and return an error value (like a `NULL` pointer).

## Specifying class properties

The `PyTypeObject` structure defines a Python type. This structure defines most of the
important attribtutes of a python object like various C functions for initializing
objects and freeing memory allocated by them when they're GC'd. It also stores a pointer
to the struct that contains the method table for the class.

Following are some important attributes:

* tp_str - Point to function implementing the built-in `str()` operator.
* tp_new - Point to function implementing instance creation function.

Links:

* https://docs.python.org/3/c-api/typeobj.html

## PyCapsule a.k.a providing C API for other extension modules

Python allows for a special way for exposing the functions of a C extension to C extensions
of other Python libraries using the `PyCapsule` constructs. It allows you encapsulate
the C API inside a Python module belonging to a particular namespace, which the other C
extension can simply load and extract the pointer to the API from.

Links:

* https://docs.python.org/2/extending/extending.html#using-capsules

## Creating Python objects from C structs

There is a macro called `PyObject_HEAD` that allows one to give Python-object like behaviour
to any C struct. It defines some variables within the struct that make it behave like
a Python object. You can then call `PyObject_GC_New` and `PyObject_GC_Track` for informing the
Python GC to keep track of these objects.

This seems very useful since it allows you to easily typecast between C structs and Python objects.

Link:

* https://docs.python.org/2/c-api/structures.html#c.PyObject_HEAD

General Links:

* https://docs.python.org/2/extending/extending.html
* https://docs.python.org/3/extending/building.html
* https://www.pythonsheets.com/notes/python-capi.html#pyobject-with-member-and-methods
* https://docs.python.org/3/c-api/index.html
