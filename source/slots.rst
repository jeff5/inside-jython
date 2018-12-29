.. File: slots.rst

Slot functions
##############

Implementation in CPython
*************************

Throughout the CPython core, you'll find large ``struct`` initialisations that look like this:

.. code-block:: c

   PyTypeObject PyFloat_Type = {
       PyVarObject_HEAD_INIT(&PyType_Type, 0)
       "float",
       sizeof(PyFloatObject),
       0,
       (destructor)float_dealloc,                  /* tp_dealloc */
       (printfunc)float_print,                     /* tp_print */
       0,                                          /* tp_getattr */
       0,                                          /* tp_setattr */
       0,                                          /* tp_compare */
       (reprfunc)float_repr,                       /* tp_repr */
       ...
       0,                                          /* tp_init */
       0,                                          /* tp_alloc */
       float_new,                                  /* tp_new */
   };

These are the initialised data of the ``type`` object (``float`` in this case).
The fields of the ``struct`` are known as "slots".
Some are descriptive data, but many
(here and in subordinate structures)
are pointers to private functions implementing operations on the objects of that type,
for example:

.. code-block:: c

   static PyObject *
   float_repr(PyFloatObject *v)
   {
       return float_str_or_repr(v, 0, 'r');
   }

When it is necessary to execute that operation,
the function in that slot is invoked:

.. code-block:: c

   PyObject *
   PyObject_Repr(PyObject *v)
   {
       ...
       res = (*Py_TYPE(v)->tp_repr)(v);
       ...
       if (!PyString_Check(res)) {
           PyErr_Format(PyExc_TypeError,
                        "__repr__ returned non-string (type %.200s)",
                        Py_TYPE(res)->tp_name);
           Py_DECREF(res);
           return NULL;
       }
       return res;
    }

As we can see,
the function in slot ``tp_repr`` provides the implementation of ``__repr__``,
for objects of that type.
Generally the special methods from the Python data model are defined in this way.

When a special method is defined in C,
as ``__repr__`` is here for ``float``,
the entry for "__repr__" in the type dictionary is an object (descriptor) that wraps the slot::

   >>> float.__repr__
   <slot wrapper '__repr__' of 'float' objects>

When a special method is defined at the Python level,
and entered into the dictionary of a type,
a C-callable wrapper is created and assigned to the corresponding slot::

   >>> class C(object):
   ...     def __repr__(self) : return "myrepr"
   ...
   >>> C.__repr__
   <unbound method C.__repr__>
   >>> C().__repr__
   <bound method C.__repr__ of myrepr>

Notice that in the last output,
in reproducing the contents of the bound method,
CPython called (through the slot) the function we defined in Python,
so that the value to which it is bound is called "myrepr".
For a plain ``object``, the result would have been::

   >>> object().__repr__
   <method-wrapper '__repr__' of object object at 0x0000000002C590C0>

In a type that does not define a given special method,
that method's slot contains a value inherited from its parent::

   >>> C.__str__
   <slot wrapper '__str__' of 'object' objects>

For the particular case of ``__str__``,
the implementation (in ``object``) calls ``__repr__``,
that is, it calls whatever is in the ``tp_repr`` slot::

   >>> C().__str__()
   'myrepr'

We may give a Python class a new definition of a special method
by assigning it as an attribute, even after definition of the class::

   >>> def f(c) : return "mystr"
   ...
   >>> C.__str__ = f
   >>> C().__str__()
   'mystr'

This attribute then appears in the dictionary of the type,
although it is not possible to make such an entry directly::

   >>> C.__dict__
   mappingproxy({'__module__': '__main__', '__repr__': <function C.__repr__ at 0x0000019BDBA679D8>,
   '__dict__': <attribute '__dict__' of 'C' objects>,
   '__weakref__': <attribute '__weakref__' of 'C' objects>,
   '__doc__': None, '__str__': <function f at 0x0000019BDB78C1E0>})
   >>> C.__dict__["__str__"] = f
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   TypeError: 'mappingproxy' object does not support item assignment

But this is not the only effect:
because the attribute name is one of the special method names,
setting the attribute also enters (a wrapper of) the function ``f`` in the C-level slot.
This is a behaviour built into ``type.__setattr__``
(in the slot ``tp_setattro`` of the type ``type``).
Built-in types do not generally permit redefinition of special functions::

   >>> float.__str__ = f
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   TypeError: can't set attributes of built-in/extension type 'float'


Approach taken by Jython
************************

Outline
=======

Jython was first developed before Java ``MethodHandle`` entered the language,
so it could provide no direct equivalent to assignable CPython slots.
(We have the choice now, but it would be a big change with user impact.)
Instead, Jython proceeds somewhat oppositely:
rather than building a data structure in the type object,
the base ``PyObject`` defines methods with the same names as the Python special methods,
``__repr__``, ``__add__``, and so on.
These act like the slots, but are expressed directly as actions (code),
rather than as values (pointers to code).

The action of the base implementation of each slot method
has to be whatever CPython would do if it came to a slot and found it empty:
usually it will raise an ``AttributeError``.

A constraint of the approach is that this type of "slot" may only be assigned
by defining a new (Java) class that extends ``PyObject``.
So how do we permit "assignment to a slot" at the Python level,
once the target class has been defined?
In the types of object where this must be allowed,
Jython looks in the type's dictionary for an attribute corresponding to the special method.
This behaviour has to be built into the definition of each special method.

The normal pattern of implementation
====================================

Any Java sub-class of ``PyObject``,
representing a particular Python type,
overrides every method corresponding to a slot that that type assigns.
The body of the method calls an implementation method specific to that type.
Jython exposes this method as a special method of the object in Python,
by marking it with a Jython-specific annotation,
through a dictionary entry (descriptor) that will call the implementation method.
For example,
here is the approach taken to define ``__repr__`` in ``float``:

.. code-block:: java

       @Override
       public PyString __repr__() {
           return float___repr__();
       }

       @ExposedMethod(doc = BuiltinDocs.float___repr___doc)
       final PyString float___repr__() {
           return Py.newString(formatDouble(SPEC_REPR));
       }

Here, ``PyObject.__repr__`` is a slot overidden by ``PyFloat``,
and ``float___repr__`` is the implementation that is both
called by the Java slot and exposed in the dictionary of ``float`` as ``__repr__``.



Java-inheritance and the ``Derived`` classes
============================================




.. code-block:: java




Code generated by exposing a method
===================================

Amongst other things, the exposer creates:

.. code-block:: text

   public class org.python.core.PyFloat$float___repr___exposer extends org.python.core.PyBuiltinMethodNarrow {
   ...
     public org.python.core.PyObject __call__();
       Code:
          0: aload_0
          1: getfield      #38 // Field self:Lorg/python/core/PyObject;
          4: checkcast     #40 // class org/python/core/PyFloat
          7: invokevirtual #44 // Method org/python/core/PyFloat.float___repr__:()Lorg/python/core/PyString;
         10: areturn
   }

Clearly this calls the method ``PyFloat.float___repr__()`` that we decorated, using ``invokevirtual``.


Exposing non-final methods
==========================

Occasionally, we find that the normal pattern of implementation is broken,
as in ``PyFloat`` (at the time of writing):

.. code-block:: java

       @ExposedMethod(doc = BuiltinDocs.float_hex_doc)
       public PyObject float_hex() {
           return new PyString(pyHexString(getValue()));
       }

       @ExposedMethod(doc = BuiltinDocs.float_as_integer_ratio_doc)
       public PyTuple as_integer_ratio() {
           // ... implementation omitted ...
           return new PyTuple(numerator, denominator);
       }

The overridable ``float_hex`` and ``as_integer_ratio`` are exposed, not a final method.
The name ``float_hex`` is what we would expect.
Notice that the exposer is able to infer the name ``hex``,
since it knows that the class is exposed as ``float``,
it knows to remove ``float_`` from the front of the name.
In the second case,
the exact Java name ``as_integer_ratio``,
since it does not begin ,
is the name of this method in the dictionary.

What difference does it make that these mathods are not final?
None to the wrapper object that appears in the dictionary:

.. code-block:: text

   public class org.python.core.PyFloat$float_hex_exposer extends org.python.core.PyBuiltinMethodNarrow {
     public org.python.core.PyObject __call__();
       Code:
          0: aload_0
          1: getfield      #38 // Field self:Lorg/python/core/PyObject;
          4: checkcast     #40 // class org/python/core/PyFloat
          7: invokevirtual #43 // Method org/python/core/PyFloat.float_hex:()Lorg/python/core/PyObject;
         10: areturn
   }

   public class org.python.core.PyFloat$as_integer_ratio_exposer extends org.python.core.PyBuiltinMethodNarrow {
     public org.python.core.PyObject __call__();
       Code:
          0: aload_0
          1: getfield      #38 // Field self:Lorg/python/core/PyObject;
          4: checkcast     #40 // class org/python/core/PyFloat
          7: invokevirtual #44 // Method org/python/core/PyFloat.as_integer_ratio:()Lorg/python/core/PyTuple;
         10: areturn
   }



The curious case of ``__str__`` and ``__repr__``
================================================

There is a particularly interesting, and fundamental,
example where the pattern of implementation is broken,
in ``PyObject`` (at the time of writing):

.. code-block:: java

       @ExposedMethod(names = "__str__", doc = BuiltinDocs.object___str___doc)
       public PyString __repr__() {
           return new PyString(toString());
       }

       @Override
       public String toString() {
           return object_toString();
       }

       @ExposedMethod(names = "__repr__", doc = BuiltinDocs.object___repr___doc)
       final String object_toString() {
           ...
           return String.format("<%s object at %s>", name, Py.idstr(this));
       }

There are three additional odd things here.
One is that the Python name of the Java ``__repr__``
(in the dictionary of the object type) will be ``__str__``.
A second is that another method not called ``object___repr__``
is exposed as ``__repr__``.
Finally,
the return type of ``object_toString``, exposed as ``__repr__``, is Java ``String``.
The exposer has to generate code to wrap this return value,
as a ``PyString`` (``str``) if it is genuinely a ``String``,
or as ``None`` if it is (impossibly here) a Java ``null``:

.. code-block:: text

   public class org.python.core.PyObject$object_toString_exposer extends org.python.core.PyBuiltinMethodNarrow {
     public org.python.core.PyObject __call__();
       Code:
          0: aload_0
          1: getfield      #38 // Field self:Lorg/python/core/PyObject;
          4: checkcast     #40 // class org/python/core/PyObject
          7: invokevirtual #44 // Method org/python/core/PyObject.object_toString:()Ljava/lang/String;
         10: dup
         11: ifnonnull     21
         14: pop
         15: getstatic     #49 // Field org/python/core/Py.None:Lorg/python/core/PyObject;
         18: goto          24
         21: invokestatic  #53 // Method org/python/core/Py.newString:(Ljava/lang/String;)Lorg/python/core/PyString;
         24: areturn
   }

