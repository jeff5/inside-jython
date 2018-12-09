# inside-jython
Notes on how Jython works

This is a project to document aspects of the Jython interpreter as I come across them. It may be unreliable in places. And it may date as things evolve.


## Building the project

This project builds with Sphinx under Python 3. Here's how it looks to the author on Windows, using PowerShell (Posh).

### Getting started

The author uses a virtual environment so that Sphinx is installed only for the project. To get `virtualenv` use the command `pip install virtualenv`. Navigate to the cloned project, then:
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


