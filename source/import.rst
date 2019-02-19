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


.. _import-magic-2.7.1:

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


.. _import-magic-2.7.0:

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


.. _import-built-in:

A built-in module (``exceptions``)
==================================

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
#. Consider whether ``fullName`` might be a Java package.

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


.. _import-module-program:

A Python module in a Python program
===================================

In a fresh interpreter session,
with an appropriately prepared path down to `mylib/a/b/c/m.py`::

>>> import sys; sys.path[0] = 'mylib'
>>> import a.b.c.m

should (and does) result in:

*  the execution of
   `mylib/a/__init__.py`,
   `mylib/a/b/__init__.py`,
   `mylib/a/b/c/__init__.py`, and
   `mylib/a/b/c/m.py`,
*  the creation of module entries in ``sys.modules`` with keys
   ``"a"``,
   ``"a.b"``,
   ``"a.b.c"``, and
   ``"a.b.c.m"``. and
*  the binding of variable ``a`` to the module ``a``.

The tortuous logic for this may be traced in `imp.java`.

The original ``import a.b.c.m`` compiles to a call to
``imp.importOne("a.b.c.m", <current frame>, -1))``
which calls the overridable built-in effectively as
``__import__("a.b.c.m", globals(), locals(), None, -1)``.
The ``None`` here is the "from-list"
(as this is not e.g. ``from a.b.c import m``),
and it means we shall eventually return the module ``a`` for binding.
The ``-1`` in the calls is the ``level`` argument, set by the compiler,
signifying a Python 2 style of search for ``a``:
first relative to the current module, then as absolute.
Action transfers now to ``import_module_level``, with essentially these arguments:
``name = "a.b.c.m"``, ``top = true``, ``modDict = globals()``, ``fromlist = None``, ``level = -1``.

The first part of the logic is in helper ``get_parent()``,
which has access to the globals of the importing module and the ``level``.
In this case, ``get_parent`` finds that the console session is in no ``__package__``
and has the module ``__name__ = "__main__"``.
``__path__`` is not set either (so it's not a package).
``"__main__"`` is not a dotted module name and ``level = -1``,
so there is no parent name to return (return ``null``).
Relative import is not possible at the top level (as we are).
By a side effect,
the ``__package__`` of the importing module is set here to ``None``.
On return to ``import_module_level`` we have
``pkgName = null`` and ``pkgMod = null``, characterising top-level import.


.. _import-module-first:

First package
-------------

Import begins with an attempt at importing the first package in the name ``"a.b.c.m"``,
at the fragment:

.. code-block:: java

        StringBuilder parentName =
                new StringBuilder(pkgMod != null ? pkgName : "");
        PyObject topMod =
                import_next(pkgMod, parentName, firstName, name, fromlist);

In the example, ``name = "a.b.c.m"`` and ``firstName = "a"``.
``import_next`` has the side-effect of adding ``"a"`` to the ``parentName`` buffer.
Within ``import_next`` we check to see if module ``a`` is already loaded in ``sys.modules``,
in which case we may return directly.
If that is not the case, we have to load ``a``.
This is attempted via a call to ``find_module(fullName, name, null)``,
where here ``fullName = name = "a"``.

``find_module`` expresses the standard Python module import logic
applied to one requested module.
We have already described (in :ref:`import-built-in`) the logic:

#. Offer the fully qualified name to each importer on ``sys.meta_path``.
#. Attempt to load the module as a built-in (a Java class).
#. Look along ``sys.path`` for a definition of the module.
#. Consider whether ``fullName`` might be a Java package.

In this case the third option is the operative one.
We put `mylib` on ``sys.path`` at the start,
and since it needs no special importer in ``sys.path_hooks``,
we land at the call:

.. code-block:: java

            ret = loadFromSource(sys, name, moduleName, Py.fileSystemDecode(p));

where ``name = moduleName = "a"`` and ``p = "mylib"``.
``sys.path`` entries like ``p`` are usually a ``PyString``, so ``p`` needs to be decoded.

``loadFromSource`` is not well named.
It will look for any of:

*  `mylib/a.py`
*  `mylib/a$py.class`
*  `mylib/a/__init__.py`
*  `mylib/a/__init__$py.class`

It will prefer the compiled versions as long as the ``Mtime`` attribute in them,
which preserves the last modified time of the source when it was compiled,
matches that of the corresponding source.
The approximate order of events in ``loadFromSource`` (for a package) is:

#. Decide that ``a`` is a package.
#. Create a ``module`` representing package ``a``
   (with ``__path__`` set to ``["mylib/a"]``).
#. Compile the source `mylib/a/__init__.py` (if necessary) to Java byte code for ``a$py``.
#. Load (and Java-initialise) the class ``a$py`` into the JVM.
#. Construct an instance of a ``a$py``
   and call its ``PyRunnable.getMain()`` to obtain a ``PyCode`` for the main body of ``a``.
#. Execute the ``PyCode`` against the module's ``__dict__`` as both ``globals()`` and ``locals()``.

A lot of this activity is the responsibility of supporting methods we do not detail here.

Notice that the ``PyCode`` is not needed (becomes garbage) once it has been executed.
The permanent results of loading are the changes made to the module ``__dict__``.
This may include the definition of Python functions and classes
that have their own ``PyCode`` objects and other data as embedded values.
After this, we return the module ``a`` from ``import_next`` into ``import_module_level``,
and this establishes the "top module" of the import.


.. _import-module-subsequent:

Subsequent packages
-------------------

In the example of ``import a.b.c.m``, 
we have imported ``a``,
but we have a long way still to go to before we reach `m.py`.
The next significant call in ``import_module_level`` is:

.. code-block:: java

            mod = import_logic(topMod, parentName,
                    name.substring(dot + 1), name, fromlist);

where ``dot`` is the position of the first dot in the full name.
``import_logic`` has responsibility for completing the import of the module chain
down to ``m``.
The signature is:

.. code-block:: java

   static PyObject import_logic(PyObject mod, StringBuilder parentName,
           String restOfName, String fullName, PyObject fromlist)

Internally ``import_logic`` loops over the elements of the module path,
loading each package,
ending with the module defined by `mylib/b/c/m.py`.
We enter with
``mod = <module 'a' from 'mylib/a/__init__.py'>``,
``parentName = "a"``,
``restOfName = "b.c.m"``,
``fullName = "a.b.c.m"``, and
``fromlist = None``.

During the iteration, ``import_next``, already discussed above, is repeatedly called, as:

.. code-block:: java

            mod = import_next(mod, parentName, name, fullName, fromlist);

and as we move down the chain from one module to the next,
``parentName`` becomes the name of the enclosing module (first time ``"a"``),
and ``name`` is the simple name of the next module sought (first time ``"b"``).

As previously,
``import_next`` looks for the target by its full name (here ``"a.b"``) in ``sys.modules``.
In this call, ``mod`` is not ``null`` as it was in the top-level, but is the module ``a``,
and so ``import_next`` will look up ``b`` via ``mod.impAttr(name.intern())``.

``PyModule.impAttr`` gets the module ``__name__``.
From this and the simple module name it deduces the full name ``"a.b"``,
and checks ``sys.modules`` for it (again).
It gets the package ``__path__`` (or makes an empty one), and
then seeks the package via essentially ``attr = imp.find_module("b", "a.b", ['mylib/a'])``.
This differs from the call ``import_next`` might have made, only in the non-null "path".

.. note::
   The actions of ``PyModule.impAttr`` appear largely to duplicate those in ``import_next``
   around where ``impAttr()`` is called.
   A check is made for the module in ``sys.modules`` duplicating the one before the call.
   ``impAttr`` checks to see if the module sought is a Java package,
   which ``import_next`` also does (although the call is to ``JavaImportHelper.tryAddPackage``).
   Any entry made in ``sys.modules`` during the import is allowed to supersede the
   result, first in ``impAttr``, then again in ``import_next``.

The search strategy of ``find_module`` has already been described
(try ``sys.meta_path``, built-in modules, the path and path hooks)
except that in this case, the parent package's ``__path__`` is used, not ``sys.path``.
There is no path hook corresponding to `mylib/a`, so ``loadFromSource``,
correctly deducing ``a.b`` to be a package,
ends up compiling and executing `mylib/a/b/__init__.py`.
The module this creates is eventually returned to the loop within ``import_logic``.

The next pass of that loop imports ``a.b.c`` by an exactly parallel process.
The import of ``a.b.c.m`` is almost the same except that the module is not a package.
This final ``import_next`` gets us out of the loop in ``import_logic``,
which returns the module ``a.b.c.m``.
However, ``import_module_level`` has remembered that we wanted the first in the chain,
ultimately because there is no "from-list",
and finally we return through ``importOne`` into the compiled instruction with
``<module 'a' from 'mylib\a\__init__.py'>`` for assignment to ``a``.

.. note::
   The Jython versions of ``get_parent`` and ``import_module_level`` differ from the CPython ones.
   The difference in signatures may not be that significant,
   representing the choice to return a ``String`` rather than update a ``char *`` buffer. 
   Differences in structure and logic may be significant. 


.. _import_relative_implicit:

A Python module by relative import
==================================

Suppose we have a module file `mylib/a/b/m3.py` like this::

   import c.m
   print "Executed: ", __file__, "c.m =", c.m

where, as previously, the path down to `mylib/a/b/c/m.py` is prepared with `__init__.py`
files to create packages ``a``, ``a.b`` and ``a.b.c``.
From the form of the import (in Python 2)
you might expect the imported module ``c.m`` to be defined either in `mylib/a/b/c/m.py`
or in `mylib/c/m.py` (or an equivalent to `mylib` elsewhere on ``sys.path``).
We know it will be found at `mylib/a/b/c/m.py`,
but the compiler didn't,
and so this has to be a run-time discovery.

In a fresh interpreter session,
with an appropriately prepared path down to `mylib/a/b/c/m.py`::

   >>> import sys; sys.path[0] = 'mylib'
   >>> import a.b.m3

The action begins with an absolute import as already studied (in :ref:`import-module-program`),
but as the body of ``m3`` is executed, during ``createFromCode``,
we reach the implicitly relative ``import c.m``.
We take up the story there.

The compiler has translated ``import c.m`` into ``imp.importOne("c.m", <current frame>, -1)``,
just as before.
Action transfers now to ``import_module_level``, with essentially these arguments:
``name = "c.m"``, ``top = true``,
``modDict = a.b.m3.globals()``,
``fromlist = None``,
``level = -1``.

The first part of the logic is in helper ``get_parent``,
which has to work out the parent module name using ``globals()`` and ``level`` as input.
``get_parent`` works out that the package name of the importing module ``a.b.m3``
is ``a.b``.
(It sets the module's ``__package__`` to this by side-effect, since it was missing.)

If ``a.b.m3`` had been a package itself, its name would be the package name.
If ``level`` had been a positive number,
signifying that the compiler saw so-many dots before the name in the import statement,
``get_parent`` would have stripped a further ``level-1`` elements from the package name.
It would be an error at this point for ``"a.b"`` not to be a key in ``sys.modules``.

From this point,
``import_module_level`` is able to call the equivalent of
``topMod = import_next(<module 'a.b'>, "a.b", "c", "c.m")`` 
which succeeds in importing ``c`` relative to ``a.b``.
Previously we encountered almost exactly this call as part of a top-level ``import a.b.c.m``,
in the loop of ``import_logic``,
so we don't need to walk through it again.
However, notice that ``c`` is going to become the head module returned by ``import_module_level``,
and ultimately returned to the body code of ``m3``,
to be assigned to ``a.b.m3.__dict__['c']``.


.. _import-module-absolute:

A Python module by absolute import
==================================

Suppose we have a module file `mylib/a/b/m4.py` like this::

   import sys
   print "Executed: ", __file__, "sys =", sys

It is perfectly obvious to any Python programmer
that we are importing the top-level ``sys`` module.
But the form of the import is exactly the same as it was for an implicit relative import.
The import mechanism has to be ready for the possibility that ``a.b.sys`` is the module intended.

In a fresh interpreter session,
with an appropriately prepared path down to `mylib/a/b/c/m.py`::

   >>> import sys; sys.path[0] = 'mylib'
   >>> import a.b.m4

The action begins with an absolute import as already studied (in :ref:`import-module-program`),
but as the body of ``m4`` is executed, during ``createFromCode``,
we reach the (possibly) implicitly relative ``import sys``.
We take up the story there.

The compiler has translated ``import sys`` into ``imp.importOne("sys", <current frame>, -1)``.
We arrive in ``import_module_level``, with essentially these arguments:
``name = "sys"``,
``top = true``,
``modDict = a.b.m4.globals()``,
``fromlist = None``,
``level = -1``.
The helper ``get_parent`` works out that
the package name of the importing module ``a.b.m4`` is ``a.b``.


.. _import-module-relative-attempt:

The relative import attempt
---------------------------

``import_module_level`` now calls the equivalent of
``topMod = import_next(<module 'a.b'>, "a.b", "sys", "sys")``,
in order to search out a module called ``a.b.sys``.
Unlike previous examples, this is not going to succeed,
and it is worth following the steps by which Jython decides no such module exists.

``import_next`` first checks ``sys.modules`` for the key ``a.b.sys``.
No such module has been imported, and so it moves on.

Because there is an enclosing module,
``import_next`` calls ``PyModule.impAttr``.
In ``impAttr``, the check in ``sys.modules`` fails (again),
so it calls effectively ``attr = imp.find_module("sys", "a.b.sys", ['mylib/a/b'])``.

Within ``find_module``,
there is nothing on ``sys.meta_path``,
and ``loadBuiltin`` doesn't find ``"a.b.sys"`` as a key in the built-in module table,
so it searches the path of the parent module.
``a.b.__path__ == ['mylib/a/b']``.
The attempt in ``loadFromSource`` fails to find any of:

*  `mylib/a/b/sys.py`
*  `mylib/a/b/sys$py.class`
*  `mylib/a/b/sys/__init__.py`
*  `mylib/a/b/sys/__init__$py.class`

and so ``find_module`` returns ``null`` to ``impAttr``.

Now, ``impAttr`` tries to find ``a.b.sys`` as a Java class:

.. code-block:: java

            attr = PySystemState.packageManager.lookupName(fullName);

This begins by looking up ``"a"`` in the nameless top-level Java package.
(``PySystemState.packageManager`` keeps a tree-like index
built by scanning the Java run-time and JARs on the class path.)
The nodes are ``PyJavaPackage`` objects (a subclass of ``PyObject``),
and so in looking up ``a`` as an attribute, we land at ``PyJavaPackage.__findattr_ex__``.
There is no matching key in the ``__dict__`` of the root ``PyJavaPackage``,
but each ``PyJavaPackage`` has
a ``PackageManager __mgr__`` member that will search for a Java package by that name,
via a call to ``__mgr__.packageExists``.

In this case ``packageExists`` enumerates the current working directory but there is no
directory `./a`.
(If there were, the manager would add it to the index.)
It then moves on to do the same for entries along ``sys.path``.
`mylib/a` exists, but contains no (non-Python) Java class files.
(Interestingly, having found `mylib/a`,
the manager does not look any further
for other places on ``sys.path`` that might be Java packages.)

``PyJavaPackage.__findattr_ex__`` then consults ``__mgr__.findClass``
to see if ``"a"`` designates a Java class (rather than a package).
Going via ``Py.findClass``, and ``Py.loadAndInitClass``,
it tries to load and initialise a class ``"a"``.
It doesn't exist, so we end up empty-handed in ``impAttr``,
then in ``import_next``:
the module ``a.b`` has no ``sys`` attribute.

``import_next`` will now try to find ``"sys"`` as a Java package
through a call that is effectively ``JavaImportHelper.tryAddPackage("sys", Py.None)``.

.. note::
   It seems odd that at this point ``outerFullName = "sys"``
   while ``fullname = "a.b.sys" is longer
   and used subsequently to see if the module got added to ``sys.modules``.

The operation of ``tryAddPackage`` in this case falls through
to looking for ``sys`` as a package in the JVM.
For this purpose it asks for a list of all packages currently known to the JVM
and builds a ``TreeMap`` with the packages,
and their containing packages (e.g. ``{java, java.lang, java.lang.invoke}``).
If ``"sys"`` were among the keys,
``tryAddPackage`` would try to add a module to ``sys.modules`` for it.

There isn't one, of course,
which finally allows ``import_next`` to conclude that there is no ``a.b.sys`` module,
and report this back into ``import_module_level``.


.. _import-module-absolute-attempt:

The absolute import attempt
---------------------------

An implicit relative import having failed,
``import_module_level`` decides that the proper ``parentName`` is an empty string.
It now calls the equivalent of
``topMod = import_first("sys", "", "sys", [])``,
in order to search out a top-level module called ``sys``.

``import_first`` delegates to ``import_next`` which quickly finds ``sys`` in ``sys.modules``.
If we had chosen in `m4.py` to import a module not already loaded,
a chain of events would unfold like that already described for :ref:`import-module-program`.


.. _import-from-module:

A Python module by ``from-import``
==================================

Suppose we have a module file `mylib/a/b/m5.py` like this::

   from a.b.c import m, n1, n2
   print "Executed: ", __file__, "m =", m, (n1, n2)

The interesting thing about this import is that
``n1`` and ``n2`` are conventional attributes of package ``a.b.c``,
while ``m`` is a module that only becomes an attribute once it is imported,
as a result of this statement.
In order to set up the expected attributes, let `mylib/a/b/c/__init__.py` be::

   print "Executed: ", __file__
   n0, n1, n2, n3, n4 = range(5)

where, as previously, the path down to `mylib/a/b/c/m.py`
is prepared with `__init__.py` files
to create packages ``a``, ``a.b`` and ``a.b.c``.

In a fresh interpreter session::

   >>> import sys; sys.path[0] = 'mylib'
   >>> import a.b.m5

The import activity as far as invoking ``m5``
is as already studied (in :ref:`import-module-program`),
but as the body of ``m5`` is executed, during ``createFromCode``,
we reach ``from a.b.c import m, n1, n2``.
We take up the action in ``import_module_level``
where ``name = "a.b.c"``, ``level = -1`` and ``fromlist = ('m', 'n1', 'n2')`` (a ``tuple``).


``a.b.c`` may be an implicit relative import.
``get_parent`` works out that
the package name of the importing module ``a.b.m5`` is ``a.b``.
Therefore our first hypothesis is that we're looking for ``a.b.a.b.c``,
and so our first call is to ``import_next`` with ``mod = "a.b"`` and ``name = "a"``.
The action unfolds as described in :ref:`import-module-relative-attempt`:

 * look for `mylib/a/b/a$py.class`,
 * look for ``a.b.a`` as a Java package,
 * search ``sys.path`` for class ``a.b.c.m`` and the rest of the from-list elements
   through the ``SysPathJavaLoader`` and thread context loader.
 * Finally we decide there is no ``a.b.a`` and are back in ``import_module_level``.

Having tried ``import_next`` first, we try ``import_first`` next.

This call is the equivalent of
``topMod = import_first("a", parentName, "a.b.c", ('m', 'n1', 'n2'))``,
where ``parentName`` is an initially empty ``StringBuilder``.
This call returns quickly with the first in the chain ``a`` as an already-imported module,
found by ``import_next``.
(The from-list seems superfluous
but is used by ``import_next`` to explore Java class possibilities.)

Compare this with :ref:`import-module-absolute-attempt`:
this time the target module is not top-level and we have a from-list.
Because here the target ``a.b.c`` is a dotted name,
``import_module_level`` will continue its descent by a call:

.. code-block:: java

      mod = import_logic(topMod, parentName, "b.c", "a.b.c", ("m", "n1", "n2"))

As we have seen,
``import_logic`` works by iterated calls to ``import_next``.

In this example, the first such looks for ``a.b`` and finds it already imported.

The second looks for ``a.b.c`` which is not yet imported.
This will go via ``PyModule.impAttr`` for the module instance ``a.b``,
in which ``imp.find_module`` will succeed in returning the module ``a.b.c``.
This is the point at which ``a.b`` gets a new attribute ``c`` referring to the module found.

The next interesting action rounds out ``import_module_level`` with a call to ``ensureFromList``.
Effectively this is called with
``mod = <module 'a.b.c'>``,
``fromlist = ('m', 'n1', 'n2')``,
``name = "a.b.c"``,
``recursive = false``.
It iterates the from-list to make sure,
by a call to ``mod.__findattr__``,
that each name on it is actually an attribute of the module.

The first name on the list ``m``, is not an attribute,
so the call ``mod.__findattr__("m")`` ends up in ``impAttr``,
and hence automatically imports ``a.b.c.m`` via a call to ``find_module``.
This import adds the module as an attribute ``m`` to ``a.b.c``.

.. note::
   This is incorrect Python behaviour, for a simple ``__findattr__`` to result in an import.
   In the code of ``ensureFromList``, if the find fails,
   ``ensureFromList`` itself will organise the import.
   The implementation of ``PyModule.impAttr`` is part of the 2.7.1 magic,
   described in :ref:`import-magic-2.7.1`.

The other two calls ``mod.__findattr__("n1")`` and ``mod.__findattr__("n2")``
find their targets as attributes in the dictionary of ``a.b.c``.

The value finally returned by ``import_module_level``,
and hence by the import operation as a whole,
is ``<module 'a.b.c' from 'mylib/a/b/c/__init__$py.class'>``.
Code generated in the caller `m5.py` receives this value into a temporary variable,
then assigns module globals ``m``, ``n1``, and ``n2`` from its attributes.

Execution of `m5.py` now continues as part of the import of that at the prompt,
and the final state is illustrated by::

   >>> a.b.m5.m, a.b.m5.n1, a.b.m5.n2
   (<module 'a.b.c.m' from 'mylib\a\b\c\m$py.class'>, 1, 2)


.. _import-from-relative-module:

A Python module by relative ``from-import``
===========================================

Suppose we have a module file `mylib/a/b/m6.py` like this::

   from ..b.c import m, n1, n2
   print "Executed: ", __file__, "m =", m, (n1, n2)

This import has the same meaning as that studied in :ref:`import-from-module`,
but expressed as a relative module reference.
(We could have written ``.c`` instead of ``..b.c``.)

The expected attributes, are set up as before in `mylib/a/b/c/__init__.py`::

   print "Executed: ", __file__
   n0, n1, n2, n3, n4 = range(5)

and, as previously, the path down to `mylib/a/b/c/m.py`
is prepared with `__init__.py` files
to create packages ``a``, ``a.b`` and ``a.b.c``.

In a fresh interpreter session::

   >>> import sys; sys.path[0] = 'mylib'
   >>> import a.b.m6

The only difference from :ref:`import-from-module`
occurs where we hit the relative import during execution of ``from ..b.c import m, n1, n2``.
The explicit relative import is going to save us searching for ``a`` relative to ``a.b``.

We take up the action in ``import_module_level``
where ``name = "b.c"``, ``level = 2`` and ``fromlist = ('m', 'n1', 'n2')`` (a ``tuple``).
Notice that ``name`` contains none of the leading dots,
but they have been counted in ``level``.
This first bites in ``get_parent``,
which deduces first that the ``__package__`` of the importing module is ``a.b``,
and then takes off ``level-1 = 1`` further elements to return ``a``.
This exists (as it must) in ``sys.modules``,
so in ``import_module_level`` we have ``pkgName = "a"`` and ``pkgMod = <module 'a'>``.

The first import is accomplished by a call that is effectively
``topMod = import_next(<module 'a'>, parentName, "b", "b.c", ('m', 'n1', 'n2'))``,
where the ``parentName`` buffer is initialised to ``pkgName = "a"``.
``a.b`` is in ``sys.modules``, so that returns quickly with it.
``topMod = <module 'a.b'>`` and ``parentName = "a.b"``.

``import_module_level`` will continue its descent by a call:

.. code-block:: java

      mod = import_logic(topMod, parentName, "b.c", "a.b.c", ("m", "n1", "n2"))

Within ``import_logic``,
the first call to ``import_next`` resolves ``a.b.c``
through a call to ``PyModule.impAttr`` on ``<module 'a.b'>``.
We return into ``import_module_level`` with ``<module 'a.b.c'>``,
and the sequel (``ensureFromList`` imports ``m``) is the same as in :ref:`import-from-module`.



A Java class in a Java package
==============================



A Java class in a Python package
================================



