.. File: modules-defined-in-java.rst

Modules Defined in Java
#######################

Introduction
************

Feature
=======

In Python, functions are objects::

   >>> def f() : pass
   ...
   >>> f
   <function f at 0x00000000033A3278>
   >>> type(f)
   <type 'function'>

And so are functions defined in modules,
even when those modules are not defined in Python::

   >>> import math
   >>> math.acos
   <built-in function acos>
   >>> math.__dict__['acos']
   <built-in function acos>
   >>> type(math.acos)
   <type 'builtin_function_or_method'>

Module dictionaries do not just contain functions::

   >>> math.pi
   3.141592653589793
   >>> type(math.pi)
   <type 'float'>

How are functions and values defined in C or Java exposed
for the interpreter to handle as Python objects?
What happens when we invoke a function?

Exposure in CPython
===================

The CPython approach is well-documented in `Extending and Embedding the Python Interpreter`_.
For modules in particular,
the extended example in `Extending Python with C or C++`_ is useful.
For comparison with Jython, we'll look at some examples from the CPython code base,
in `mathmodule.c`:

.. code-block:: c

   static PyMethodDef math_methods[] = {
       {"acos",            math_acos,      METH_O,         math_acos_doc},
       {"acosh",           math_acosh,     METH_O,         math_acosh_doc},
       ...
       {"trunc",           math_trunc,     METH_O,         math_trunc_doc},
       {NULL,              NULL}           /* sentinel */
   };

   PyMODINIT_FUNC
   initmath(void)
   {
       PyObject *m;

       m = Py_InitModule3("math", math_methods, module_doc);
       if (m == NULL)
           goto finally;

       PyModule_AddObject(m, "pi", PyFloat_FromDouble(Py_MATH_PI));
       PyModule_AddObject(m, "e", PyFloat_FromDouble(Py_MATH_E));

       finally:
       return;
   }

Elsewhere in `mathmodule.c`
CPython defines ``math_acos`` (and all the parallel functions)
to be the corresponding system maths library function (``acos()``),
in a wrapper that detects and reports error conditions.
This is not very clear as it is constructed using a macro.

Observe in the snippet above that a module initialisation utility ``Py_InitModule3``
consumes a table using pointers to function.
Each row describes a function object constructed by ``Py_InitModule3`` for that function,
with type ``builtin_function_or_method``,
and entered in the module dictionary.
The constant ``METH_O`` signals that the function takes a single object as argument.
It sets a property that ultimately determines how the function is called (in `ceval.c`)
when invoked from Python.

Observe also that the constants ``e`` and ``pi``
(which were not in the table)
are added to the dictionary "by hand",
within the module initialisation function.

.. _Extending and Embedding the Python Interpreter:
   https://docs.python.org/2.7/extending/index.html#extending-and-embedding-the-python-interpreter

.. _Extending Python with C or C++:
   https://docs.python.org/2.7/extending/extending.html

.. _The Module’s Method Table and Initialization Function:
   https://docs.python.org/2.7/extending/extending.html#the-module-s-method-table-and-initialization-function



Approaches used in Jython
*************************

Reflection
==========

A module may be defined for Jython by defining a Java class,
and listing the module and its class in ``org.python.modules.Setup.builtinModules``.
As a result of the listing,
the import machinery will recognise the module name ("math" for example),
and eventually return the result of ``org.python.core.PyType.fromClass(Class<?>)``
on the defining class (``org.python.module.math`` in the example)::

   import math

This has the unexpected effect of making the module a ``type`` in Jython,
rather than a ``module``.
(This might be a defect.)

The mechanism within ``PyType.fromClass()`` behaves as it would for any Java class,
and as a result:

#. ``public static`` methods are entered in the dictionary of the module
   as function-descriptor attributes of the module.
#. ``public static`` fields are entered in the dictionary of the module
   as data-descriptor attributes of the module.

The effect of this (in a Jython console) is:

.. code-block:: python

   >>> math.acos
   <java function acos 0x84>
   >>> math.pi
   3.141592653589793
   >>> type(math.__dict__['acos'])
   <type 'reflectedfunction'>
   >>> type(math.__dict__['pi'])
   <type 'reflectedfield'>

Jython is slightly different from Python here, in having as dictionary entries,
Python objects that visibly refer to Java members, by reflection,
rather than fully disguising the functions and data as Python values.

In fact,
the mechanism within ``PyType.fromClass()`` does some extra work for a Java module
compared to what it would for a plain Java class,
if the Java class declares that it implements ``ClassDictInit``.
Such a class must then define ``classDictInit(PyObject)``,
which ``PyType.fromClass()`` will call
(actually in ``PyJavaType.init()``)
allowing the module to customise its dictionary.
The one in ``org.python.modules.math`` is empty,
but the one in ``org.python.modules._io._jyio`` runs:

.. code-block:: java

       public static void classDictInit(PyObject dict) {
           dict.__setitem__("__name__", new PyString("_jyio"));
           dict.__setitem__("__doc__", new PyString(__doc__));
           dict.__setitem__("DEFAULT_BUFFER_SIZE", DEFAULT_BUFFER_SIZE);

           dict.__setitem__("_IOBase", PyIOBase.TYPE);
           dict.__setitem__("_RawIOBase", PyRawIOBase.TYPE);
           dict.__setitem__("FileIO", PyFileIO.TYPE);
           // ...
           // Hide from Python
           dict.__setitem__("classDictInit", null);
           dict.__setitem__("makeException", null);
       }

In this we see additions to and removals from the dictionary,
and also how a module can be made to contain other types (that is, to export classes).


Hand-crafting (``__builtin__`` module)
======================================

The ``__builtin__`` module,
which is what the module variable ``__builtins__`` usually points to,
is made a special case in Jython,
apparently for speed in the invocation of these frequently-used functions.



