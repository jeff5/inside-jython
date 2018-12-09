# inside-jython
Notes on how Jython works

This is a project to document aspects of the Jython interpreter
as I come across them.
It may be unreliable in places.
And it may date as things evolve.


## Building the project

This project builds with Sphinx under Python 3.
Here's how it looks on Windows, using PowerShell (Posh).

### Getting started

A virtual environment is recommended
so that Sphinx is installed only for the project.
To get `virtualenv` use the command `pip install virtualenv`.
Navigate to the cloned project, then:
```
PS inside-jython> virtualenv venv
Using base prefix ...
New python executable in ...\venv\Scripts\python.exe
Installing setuptools, pip, wheel...
done.
PS inside-jython> .\venv\Scripts\activate
(venv) PS inside-jython> pip install sphinx
Collecting sphinx
  Downloading ... (loads of stuff)
(venv) PS inside-jython>
```

### Building to test

The root of the project contains a script `make.bat`that builds the project.
Sphinx is able to produce several output formats,
but the easiest one in which to browse is HTML.
It looks something like this:
```
(venv) PS inside-jython> .\make html
Running Sphinx v1.8.2
making output directory...
loading intersphinx inventory from https://docs.python.org/objects.inv...
intersphinx inventory has moved: https://docs.python.org/objects.inv -> https://docs.python.org/3/objects.inv
building [mo]: targets for 0 po files that are out of date
building [html]: targets for 1 source files that are out of date
updating environment: 1 added, 0 changed, 0 removed
reading sources... [100%] index
looking for now-outdated files... none found
pickling environment... done
checking consistency... done
preparing documents... done
writing output... [100%] index
generating indices... genindex
writing additional pages... search
copying static files... done
copying extra files... done
dumping search index in English (code: en) ... done
dumping object inventory... done
build succeeded.
```
Then visiting `build/html/index.html` with a browser
allows one to view the documentation. 

