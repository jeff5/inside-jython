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

Basic behaviour of Jython
=========================

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

A Python package ``a.b.c`` that also defines classes in Java
would have as attributes only the objects defined in `__init__.py`
and the Java classes intended by the programmer.
The programmer's intention to include them is expressed by putting those classes
alongside the `__init__.py` that defines the package.
(The package names in the class files had better match too.)
Ideally,
`__init__.py` would be able to control the visibility of the Java classes,
by standard mechanisms such as ``__all__``.
For this to work,
those classes would have to be in the global dictionary of the module as it initialised.
The rule concerning names beginning with an underscore would apply to Java classes during import.

A Python module within that Python package has to be imported in a separate ``import`` statement,
as Python requires,
and only then becomes an attribute of the package.
``a.b.c.m`` will often be defined by an `a/b/c/m.py`,
either `a/b/c/m.py` alongside `a/b/c/__init__.py`,
or elsewhere on ``a.b.c.__path__``.

Suppose, ``a.b.c.m`` is also (the name of) a Java package.
This expression cannot designate the ``javapackage`` and the Python ``module`` at the same time.
Once creation of a Python package (a ``PyModule``) correctly recognises Java classes it contains,
it seems unnecessary to have two kinds of module at the Python level.
Python 3 documentation says
"Python has only one type of module object,
and all modules are of this type,
regardless of whether the module is implemented in Python, C, or something else".

Ideally then,
the type ``javapackage`` would not be exposed,
but rather a Java package would appear as a ``module``.
In that case, ``a.b.c.m``, where there is no `m.py`,
would still designate a Python ``module``,
but there would be no Python code to execute on initialisation.



Sequence of events in Jython 2.7.1
**********************************

The following notes are from studying the operation of import mechanisms.

A built-in module (``exeptions``)
=================================

The import of ``exceptions`` occurs naturally during the initialisation of ``PySytemState``.
Unlike ``sys`` or ``__builtins__`` it is a regular built-in module.
Action begins with a call to ``org.python.core.Py.initClassExceptions(PyObject)``:

.. code-block:: java

       static void initClassExceptions(PyObject dict) {
           PyObject exc = imp.load("exceptions");

``imp.load`` takes the lock which protects the import system from concurrent modification,
and calls ``import_first(name, new StringBuilder())``.
This is the short form of ``import_first``.
A longer form supports ``from ... import ...``,
but both call ``import_next`` to get their work done.

In ``import_next`` we at last encounter some real import logic:

#. Check for the module (by its fully-qualified name) in ``sys.modules``.
#. Try to load the module via ``find_module``
   or find it as an attribute of a parent module via its ``impAttr`` method.
#. Try to load the module as a Java package.

The first that succeeds here gives us our result
(so if ``exceptions`` had already been imported into the Python interpreter,
we would stop at the first).
In our case, ``exceptions`` has no parent, and ``find_module`` will succeed.

``find_module`` also contains some important logic:

#. Offer the fully qualified name to each importer on ``sys.meta_path``.
#. Attempt to load the module as a built-in (a Java class).
#. Look along ``sys.path`` for a definition of the module.

``exceptions`` is a built-in module,
so the second option will find it for us using ``loadBuiltin``.

This is fairly straightforward,
since initialisation in ``Setup`` has already created a map ``PySystemState.builtins``,
from the fully-qualified name of each built-in module
to a class name for its implementation.
``exceptions`` is a key in that map, of course,
for ``"org.python.core.exceptions"``.

``loadBuiltin``  has only to load and initialise that Java class,
via ``Py.findClassEx``.
This is not quite aa simple as it looks,
since ``Py.findClassEx`` has a choice of class loaders.
The normal choice seems to be to load via ``org.python.core.SyspathJavaLoader``.

The last step is to register that class
(since it is a ``PyObject``)
as a Python type through a call to ``PyType.fromClass(c)``.
This type object is what we ultimately return from ``imp.load``.

On the way out ``import_next`` is responsible for posting this in ``sys.modules``,
against the key ``"exceptions"``::

   >>> import sys
   >>> sys.modules["exceptions"]
   <type 'exceptions'>

This is slightly at variance with CPython
where it shows as ``<module 'exceptions' (built-in)>`` and is definitely of type ``module``.


A Python module in a Python package
===================================





A Java class in a Java package
==============================


A Java class in a Python package
================================



