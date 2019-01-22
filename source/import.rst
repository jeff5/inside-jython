.. File: import.rst

Import
######

The Python ``import`` statement is the normal way in which
a main program or a module gains access to objects defined in some other module.
Information on module import is somewhat scattered
in Python 2 documentation,
and was written as the CPython reference implementation grew
over many years, and several PEPs.
It has support for certain special cases built in, and hooks for user extension.

Python 3 documentation is clearer on the required semantics,
and the implementation is more regular,
but it is different in important ways from Python 2.

Jython reproduces Python 2 import, but has some special features related to Java.


Differences from CPython
************************

CPython allows a module to be defined in a C extension.
It will treat a shared library (or DLL) as the definition of a module.
Jython does not support extensions in C.

Jython is able to import a Java package or class as a Python module.




Treatment of Java packages and classes
**************************************

General objective
=================

Jython allows us to use Java classes as types in Python programs.
The fully-qualified (dotted) name of a Java package or class
closely resembles to that of a Python package or module,
both where it occurs in a Python (or Java) import statement,
and when we refer to it.
Both languages allow for binding just the short name,
in Python by the form ``from a.b.c import D``,
and in Java because ``import a.b.c.D`` does that automatically.
It is natural to use what seems to be a common idiom,
and to expect it to mean the same thing.

There are some differences, however.

In Python, when we import a package or module, doing so initialises it,
and all its ancestor packages.
In Java we cannot import or initialise a package,
and the import of a class merely declares its name.
In Java a class is initialised on first use.

In Python, the import makes the top-level package appear in our namespace.
Thus ``import a.b.c`` initilises ``a``, ``a.b`` and ``a.b.c``,
and binds a variable ``a`` to package ``a``.
In Java, a package is merely a name grouping classes, not an object,
whereas in Python both are objects.

In Python, ``a`` is a package only if the special file `a/__init__.py`,
or its compiled form,
exists and this file is executed to initialise an object ``a``.
If it does not, for all that `a/b.py` may exist in the flesystem,
``import a.b`` will fail.
Java packages contain no such marker, yet Jython recognises them as packages.

In Python ``import a.b.c.D`` expects ``D`` to be a module (which may be a package too).
If ``a.b.c`` is a Java package,
then ``import a.b.c.D`` in Jython may be expected to make ``D`` a module,
but this is not so.
``D`` may indeed be intended as a Jython module,
but it will appear as an ordinary class within ``a.b.c``.
(Examine for example the exact type of the ``math`` module.)

In Python ``import a.b.c`` could contain an object ``D``
(a class or anything else)
accessible after ``import a.b.c`` as the expression ``a.b.c.D``.
However, if ``D`` is a module, it should not be accessible until explicitly imported.
Indeed, ``a.b`` is only meaningful because ``a.b`` is imported on the way to ``a.b.c``.

Root bahaviour of Jython
========================

In order to deal with these differences,
when Jython imports a Java package,
it represents it as a particular type ``javapackage``, distinct from ``module``.
It has different behaviour from ``module``.

Importing a ``javapackage`` effectively enumerates the classes and packages within it.
(This enumeration may in practice be read from a cache, after the first time.)
A plain import statement binds the first name in the path,
and if this is a ``javapackage``,
that name is bound to a structure navigable by the dotted names as attributes,
leading to successive packages and classes.
Thus ``import java.lang`` binds ``java``
to a structure in which ``java.nio.file.FileSystem`` is immediately available.

A problem arises when a given dotted name,
defines both a Java package and a Python package.
The package may be realised as a sequence of directories,
somewhere on the module search paths,
which is made a package by the presence of `__init__.py` modules,
but also contains Java `.class` files.
Which of the two behaviours should apply:
that of ``module`` or that of ``javapackage``?
The choice has been made and reverted several times.


Behaviour of Jython 2.0, 2.5.2, 2.7.1
=====================================

In these versions of Jython,
in order to make ``D`` accessible as ``a.b.c.D`` after ``import a.b.c``,
when ``a.b.c`` is a Python package,
the attribute access method of ``PyModule`` is modified.
When ``a.b.c.D`` is evaluated,
when Jython looks up attribute ``D`` on ``a.b.c``,
it tries to import ``a.b.c.D``.
If a file `a/b/c/D.class` exists,
that class is loaded and becomes the meaning of ``a.b.c.D``.
At the Python level, ``D`` is a class (a type) defined within package ``a.b.c``.
However, 
if `a/b/c/D.py` exists instead,
that module is loaded and becomes the meaning of ``a.b.c.D``,
which is incorrect without explicit import.


Behaviour of Jython 2.5.0, 2.5.3 and 2.7.0
==========================================

In these versions the project has removed "the magic that automatically imports a module".
The Python behaviour is now correct with these mixed directories,
but those users who find it convenient to put related Python modules
and Java classes in a single namespace
are not able to find the Java classes
(and regard this as a regression).


Behaviour proposed for 2.7.2
============================

Ideally,
a Python package that also defines classes in Java would have as attributes
only the objects defined in `__init__.py`
and the Java classes.
Ideally also,
`__init__.py` would be able to control the visibility of the Java classes,
by standard mechanisms such as ``__all__``.
(For this to work,
those classes would have to be in the global dictionary of the module as it initialised.)
The rule concerning names beginning with an underscore would apply during import.

Finally, 
if creation of a Python package (a ``PyModule``) correctly recognises Java classes it contains,
it seems unnecessary to have two kinds of module at the Python level.
So ideally again,
the type ``javapackage`` would not be exposed,
but rather a Java package would appear as a ``module``.
Python 3 documentation says
"Python has only one type of module object,
and all modules are of this type,
regardless of whether the module is implemented in Python, C, or something else.".


Sequence of events in Jython 2.7.1
**********************************


A built-in module (``exeptions``)
=================================


A Python module in a Python package
===================================


A Java class in a Java package
==============================


A Java class in a Python package
================================



