#Building and Breaking a Python Sandbox
###Building a Sandbox:
####Two main ways in python
1. Language Level Sandboxing using pysandbox
2. OS-Level Sandboxing using PyPy's Sandbox

####We would be covering Language Level Sandboxing using pysandbox in this document.

###Question : How to execute arbitrary python code in the first place?
####Answer:
1. eval : compiles and evaluates expressions
```python
eval("1+2")
```
2. exec : compiles and evaluates statements
```python
exec print("Hello World!")
Hello World!
```


##Lets Begin! 
###Create a sandbox.py file that contains our sandboxing class. Now this will be a sandbox that still executes "EVERYTHING!".
####Sandbox.py
```python
class Sandbox(object):
    def execute(self,code_string):
        exec code_string
```

###Now! Let us test it! 
####test_sandbox.py
```python
from sandbox import Sandbox
s = Sandbox()
code = """
  print("Hello World!")
  """
s.execute(code)
```

```
   $ python test_sandbox.py
     Hello World!
```

###What are the things that we should disallow?
* Untrusted User Code
* Resource Exhaustion
* Running Unexpected Services
* Disabling/quitting/erroring out of the Sandbox

Definitely! we would not want our users to be able to use up all/any of our file descriptors, 
or maybe disallow them from starting a web server... that does something (Hope you understand),
or maybe disallow errors move out of the sandbox, or maybe disallow them from reading sensitive
sensitive file information. Whatever it might be, a secure solution must be found out.

Let us consider a basic example by taking the ability to write to the file system, one of the
basic things that a sandbox should not allow. So we would be focusing on the ability to write 
to a file.

####Let us have a look at this code below:
```python
from sandbox import Sandbox
s = Sandbox()
code = """
  file("test.txt","w").write("BAAAAAAMMMMMM!! I'm Iron Man!\\n")
  """
s.execute(code)
```

####So currently this will execute, very easily. But how could we prevent it?

###What is file?
If you try to execute ```python __builtins__.__dict__.keys() ``` in your python prompt,
it will take you to :
```
>>> __builtins__.__dict__.keys()
['bytearray', 'IndexError', 'all', 'help', 'vars', 'SyntaxError', 'unicode',
'UnicodeDecodeError', 'memoryview', 'isinstance', 'copyright', 'NameError',
'BytesWarning', 'dict', 'input', 'oct', 'bin', 'SystemExit', 'StandardError', 'format', 'repr',
'sorted', 'False', 'RuntimeWarning', 'list', 'iter', 'reload', 'Warning', '__package__',
'round', 'dir', 'cmp', 'set', 'bytes', 'reduce', 'intern', 'issubclass', 'Ellipsis', 'EOFError',
'locals', 'BufferError', 'slice', 'FloatingPointError', 'sum', 'getattr', 'abs', 'exit', 'print',
'True', 'FutureWarning', 'ImportWarning', 'None', 'hash', 'ReferenceError', 'len',
'credits', 'frozenset', '__name__', 'ord', 'super', '_', 'TypeError', 'license',
'KeyboardInterrupt', 'UserWarning', 'filter', 'range', 'staticmethod', 'SystemError',
'BaseException', 'pow', 'RuntimeError', 'float', 'MemoryError', 'StopIteration',
'globals', 'divmod', 'enumerate', 'apply', 'LookupError', 'open', 'quit', 'basestring',
'UnicodeError', 'zip', 'hex', 'long', 'next', 'ImportError', 'chr', 'xrange', 'type', '__doc__',
'Exception', 'tuple', 'UnicodeTranslateError', 'reversed', 'UnicodeEncodeError',
'IOError', 'hasattr', 'delattr', 'setattr', 'raw_input', 'SyntaxWarning', 'compile',
'ArithmeticError', 'str', 'property', 'GeneratorExit', 'int', '__import__', 'KeyError',
'coerce', 'PendingDeprecationWarning', 'file', 'EnvironmentError', 'unichr', 'id',
'OSError', 'DeprecationWarning', 'min', 'UnicodeWarning', 'execfile', 'any', 'complex',
'bool', 'ValueError', 'NotImplemented', 'map', 'buffer', 'max', 'object', 'TabError',
'callable', 'ZeroDivisionError', 'eval', '__debug__', 'IndentationError',
'AssertionError', 'classmethod', 'UnboundLocalError', 'NotImplementedError',
'AttributeError', 'OverflowError']

```

So as we can see that ***file*** is in there, hence file is one of the builtins in python. Among other
potentially problematic builtins are ***exit,open,quit,file,execfile,eval***, etc.

###How do we disallow execution of problematic builtins?
####Idea : keyword blacklist
So here we would be creating a blacklist of the builtins that should not be accessible and then pass on the 
code with filtering.
```python
class Sandbox(object):
 def execute(self, code_string):
     keyword_blacklist = ["file", "quit", "eval", "exec"]
     for keyword in keyword_blacklist:
        if keyword in code_string:
            raise ValueError("Blacklisted")
     exec code_string
```

Now when we test this code, indeed it should give us an error.
```python
from sandbox import Sandbox
s = Sandbox()
code = """
file("test.txt", "w").write("Kaboom!\\n")
"""
s.execute(code)


>>>$ python test_sandbox.py
Traceback (most recent call last):
 File "test_sandbox.py", line 11, in
<module>
 s.execute(code)
 File "/Users/jesstess/Desktop/sandbox/
sandbox.py", line 86, in execute
 raise ValueError("Blacklisted")
ValueError: Blacklisted
```

###Circumventing a blacklist
####Idea : encryption
```python
func = __builtins__["file"]
func("test.txt", "w").write("Kaboom!\n")
```
####decrypting to:
```python
func = __builtins__["svyr".decode("rot13")]
func("test.txt", "w").write("Kaboom!\n")
```
####Observation: 
If I can get a reference to something bad, I can invoke it.

###How can we remove all references to problematic builtins?
####Idea : builtins whitelist
something like:
```
builtins_whitelist = set((
 # exceptions
 'ArithmeticError', 'AssertionError', 'AttributeError',
 ...
 # constants
 'False', 'None', 'True',
 ...
 # types
 'basestring', 'bytearray', 'bytes', 'complex', 'dict',
 ...
 # functions
 '__import__', 'abs', 'all', 'any', 'apply', 'bin', 'bool',
 ...
 # block: eval, execfile, file, quit, exit, reload, etc.
))
```
before we could execute the code, we could take these original builtins and then iterate over
the original builtins over keys and if it is not whitelist, we will delete it.So it will not 
be there in the namespace anymore.
```python
import sys
main = sys.modules["__main__"].__dict__
orig_builtins = main["__builtins__"].__dict__
builtins_whitelist = set(( ... ))
for builtin in orig_builtins.keys():
    if builtin not in builtins_whitelist:
        del orig_builtins[builtin]
```

testing the builtins whitelist:
```python
from sandbox import Sandbox
s = Sandbox()
code = """
file("test.txt", "w").write("Kaboom!\\n")
"""
s.execute(code)

>>>$ python test_sandbox.py
Traceback (most recent call last):
 File "test_sandbox.py", line 9, in
<module>
 s.execute(code)
 ...
 File "<string>", line 2, in <module>
NameError: name 'file' is not defined
```

###Circumventing a whitelist
####Circumvention Idea : import something dangerous
try importing a dangerous library such as:

```python
from sandbox import Sandbox
s = Sandbox()
code = """
import os
fd = os.open("test.txt", os.O_CREAT|os.O_WRONLY)
os.write(fd, "Kaboom!\\n")
"""
s.execute(code)
```

Again, we got out of the sandbox.

###How do we disallow problematic imports?
####Idea : import whitelist
But wait, How does importing a module work in Python?

__builtins__ already has the __import__ in it. So what if we disable these kinds of imports,
create our own reference variable and then load our imports from that variable through our
whitelist.

```python
>>> importer = __builtins__.__dict__.get("__import__")
>>> os = importer("os")
>>> os
<module 'os' from '/Library/Frameworks/
Python.framework/Versions/2.7/lib/python2.7/os.pyc'>
>>> os.getcwd()
'/Users/jesstess/Desktop/sandbox'

```
####What is the expected function signature for the importer?
```python
>>> help(__builtins__.__dict__["__import__"])
__import__(...)
 __import__(name, globals={}, locals={},
fromlist=[], level=-1) -> module
```

####Cool, so lets write our own importer:
```python
>>> def my_importer(module_name, globals={},
... locals={}, fromlist=[],
... level=-1):
... print "Using my importer!"
... return __import__(module_name, globals,
... locals, fromlist, level)
...
>>> os = my_importer("os")
Using my importer!
>>> os.getcwd()
'/Users/karmacoder/Desktop/sandbox'
```

Now, before we actually import something, we should first check whether it is in the whitelist or not.

```python

def _safe_import(__import__, module_whitelist):
    def safe_import(module_name, globals={}, locals={}, fromlist=[], level=-1):
        if module_name in module_whitelist:
            return __import__(module_name, globals, locals, fromlist, level)
        else:
            raise ImportError("Blocked import of %s" ( module_name,))
    return safe_import
```
We replace our importer that is in builtins with our safe import implementations;
```python
import sys
main = sys.modules["__main__"].__dict__
orig_builtins = main["__builtins__"].__dict__
for builtin in orig_builtins.keys():
    if builtin not in builtins_whitelist:
        del original_builtins[builtin]
safe_modules = ["string", "re"]
orig_builtins["__import__"] = _safe_import(__import__, safe_modules)
 ```
 
 
####Testing the safe importer:
```python
from sandbox import Sandbox
s = Sandbox()
code = """
import os
fd = os.open("test.txt", os.O_CREAT|os.O_WRONLY)
os.write(fd, "Kaboom!\\n")
"""
s.execute(code)


>>>$ python test_sandbox.py
Traceback (most recent call last):
 File "test_sandbox.py", line 11, in
<module>
 ...
 raise ImportError("Blocked import of
%s" % (module_name,))
ImportError: Blocked import of os
```
###Circumventing the safe importer
####Circumvention Idea : modifying builtins
Idea: make builtins read-only
So, How can we make an object read-only in Python? 

As we know, that builtins is a dictionary, we can subclass a dictionary, so what if we raise a
ValueError for all of the things that would try to mutate the dictionary.
```python
class ReadOnlyBuiltins(dict):
    def __delitem__(self, key):
        ValueError("Read-only!")
    def pop(self, key, default=None):
        ValueError("Read-only!")
    def popitem(self):
        ValueError("Read-only!")
    ...
    def setdefault(self, key, value):
        ValueError("Read-only!")
    def __setitem__(self, key, value):
        ValueError("Read-only!")
    def update(self, dict, **kw):
        ValueError("Read-only!")
```

So now, as we are representing it as a class, now we cannot add or remove items from this dictionary. 
Hence, before executing any untrusted code, we can replace our original builtins with our read-only version.

```python
main = sys.modules["__main__"].__dict__
orig_builtins = main["__builtins__"].__dict__
for builtin in orig_builtins.keys():
    if builtin not in builtins_whitelist:
        del original_builtins[builtin]
safe_modules = ["string", "re"]
orig_builtins["__import__"] = _safe_import(__import__, safe_modules)
safe_builtins = ReadOnlyBuiltins(original_builtins)
main["__builtins__"] = safe_builtins
```

###Observation redux: If I can get a reference to something bad, I can invoke it.
Now, Let us think and try something else that we can do to break the sandbox.
Our hacker asks himself/herself, If I can get a reference to something bad, I can invoke it,
so if not here, can I get a reference to something bad, anywhere else in the entire python?

Hence he/she found out this.

###Circumvention Idea : exploiting the inheritance hierarchy
####What can we find out about an object’s base classes?
```python
>>> dir([])
['__add__', '__class__', '__contains__',
'__delattr__', '__delitem__',
'__delslice__', '__doc__', '__eq__',
'__format__', '__ge__', ...]
>>> [].__class__
<type 'list'>
```

now the __class__ has its own __classes__.__bases__ which refers to what the object's base classes are. 
```python
>>> [].__class__
<type 'list'>
>>> [].__class__.__bases__
(<type 'object'>,)
>>> [].__class__.__bases__[0]
<type 'object'>
```

####What can we find out about an object’s subclasses?
We can also look into the subclasses of the object
```python
>>> [].__class__.__subclasses__()
[]
>>> int.__subclasses__()
[<type 'bool'>]
>>> basestring.__subclasses__()
[<type 'str'>, <type 'unicode'>]
```

So, you can find what base classes a class has, and you can also find the reverse too.
We can get the base classes of an object, and we can get the subclasses of an object, so, the base
class of a list is object, which everything inherits from, but, what are the subclasses of object?
Sounds pretty interesting to find out.

```python
>>> [].__class__.__bases__
(<type 'object'>,)
>>> [].__class__.__bases__[0]
<type 'object'>
>>> [].__class__.__bases__[0].__subclasses__()
[<type 'type'>, <type 'weakref'>, <type 'weakcallableproxy'>, <type 'weakproxy'>, <type 'int'>, <type
'basestring'>, <type 'bytearray'>, <type 'list'>, <type 'NoneType'>, <type 'NotImplementedType'>, <type
'traceback'>, <type 'super'>, <type 'xrange'>, <type 'dict'>, <type 'set'>, <type 'slice'>, <type
'staticmethod'>, <type 'complex'>, <type 'float'>, <type 'buffer'>, <type 'long'>, <type 'frozenset'>,
<type 'property'>, <type 'memoryview'>, <type 'tuple'>, <type 'enumerate'>, <type 'reversed'>, <type
'code'>, <type 'frame'>, <type 'builtin_function_or_method'>, <type 'instancemethod'>, <type 'function'>,
<type 'classobj'>, <type 'dictproxy'>, <type 'generator'>, <type 'getset_descriptor'>, <type
'wrapper_descriptor'>, <type 'instance'>, <type 'ellipsis'>, <type 'member_descriptor'>, <type 'file'>,
<type 'PyCapsule'>, <type 'cell'>, <type 'callable-iterator'>, <type 'iterator'>, <type 'sys.long_info'>,
<type 'sys.float_info'>, <type 'EncodingMap'>, <type 'fieldnameiterator'>, <type 'formatteriterator'>,
<type 'sys.version_info'>, <type 'sys.flags'>, <type 'exceptions.BaseException'>, <type 'module'>, <type
'imp.NullImporter'>, <type 'zipimport.zipimporter'>, <type 'posix.stat_result'>, <type
'posix.statvfs_result'>, <class 'warnings.WarningMessage'>, <class 'warnings.catch_warnings'>, <class
'_weakrefset._IterationGuard'>, <class '_weakrefset.WeakSet'>, <class '_abcoll.Hashable'>, <type
'classmethod'>, <class '_abcoll.Iterable'>, <class '_abcoll.Sized'>, <class '_abcoll.Container'>, <class
'_abcoll.Callable'>, <class 'site._Printer'>, <class 'site._Helper'>, <type '_sre.SRE_Pattern'>, <type
'_sre.SRE_Match'>, <type '_sre.SRE_Scanner'>, <class 'site.Quitter'>, <class 'codecs.IncrementalEncoder'>,
<class 'codecs.IncrementalDecoder'>, <class 'string.Template'>, <class 'string.Formatter'>, <type
'operator.itemgetter'>, <type 'operator.attrgetter'>, <type 'operator.methodcaller'>, <type
'collections.deque'>, <type 'deque_iterator'>, <type 'deque_reverse_iterator'>, <type
'itertools.combinations'>, <type 'itertools.combinations_with_replacement'>, <type 'itertools.cycle'>,
<type 'itertools.dropwhile'>, <type 'itertools.takewhile'>, <type 'itertools.islice'>, <type
'itertools.starmap'>, <type 'itertools.imap'>, <type 'itertools.chain'>, <type 'itertools.compress'>, <type
'itertools.ifilter'>, <type 'itertools.ifilterfalse'>, <type 'itertools.count'>, <type 'itertools.izip'>,
<type 'itertools.izip_longest'>, <type 'itertools.permutations'>, <type 'itertools.product'>, <type
'itertools.repeat'>, <type 'itertools.groupby'>, <type 'itertools.tee_dataobject'>, <type 'itertools.tee'>,
<type 'itertools._grouper'>, <type '_thread._localdummy'>, <type 'thread._local'>, <type 'thread.lock'>,
<class 'sandbox.Protection'>, <type 'resource.struct_rusage'>, <class 'sandbox.config.SandboxConfig'>,
<class 'sandbox.proxy.ReadOnlySequence'>, <class 'sandbox.sandbox_class.Sandbox'>, <class
'sandbox.restorable_dict.RestorableDict'>]
```
Whhooooaaaaaahhhhhh!!!!!!!
So these are all that subclass the object class. So this is pretty tough to read, so let us create a for loop
and break it down.
```python
>>> obj_class = [].__class__.__bases__[0]
>>> for c in obj_class.__subclasses__():
... print c.__name__
...
wrapper_descriptor
instance
ellipsis
member_descriptor
file
PyCapsule
cell
callable-iterator
iterator
...
```


Did you just happen to notice the file tag:
... print c.__name__
...
wrapper_descriptor
instance
ellipsis
member_descriptor
***file***
PyCapsule
cell
callable-iterator
iterator
...




Now, if we try and test our circumvention:
```python
from sandbox import Sandbox
s = Sandbox()
code = """
obj_class = [].__class__.__bases__[0]
obj_subclasses = dict((elt.__name__, elt) for \
 elt in obj_class.__subclasses__())
func = obj_subclasses["file"]
func("text.txt", "w").write("Kaboom!\\n")
"""
s.execute(code)
```

Our sandbox just got broken.

###Idea : Don’t expose dangerous implementation details.
Let’s delete __bases__ and __subclasses__
```python
>>> type.__bases__
(<type 'object'>,)
>>> del type.__bases__
>>> type.__bases__
(<type 'object'>,)
>>> del type.__bases__
Traceback (most recent call last):
 File "<stdin>", line 1, in <module>
TypeError: can't set attributes of builtin/extension
type 'type'
```
Whaattt??? The underlying CPython implementation does not allow you to do that at all.
Gladly, we have a workaround.

####cpython.py
```python
from ctypes import pythonapi, POINTER, py_object
_get_dict = pythonapi._PyObject_GetDictPtr
_get_dict.restype = POINTER(py_object)
_get_dict.argtypes = [py_object]
del pythonapi, POINTER, py_object
def dictionary_of(ob):
    dptr = _get_dict(ob)
    return dptr.contents.value
```

Now in our main program file:
```python
from cpython import dictionary_of
main = sys.modules["__main__"].__dict__
...
safe_builtins = ReadOnlyBuiltins(original_builtins)
main["__builtins__"] = safe_builtins
type_dict = dictionary_of(type)
del type_dict["__bases__"]
del type_dict["__subclasses__"]
```

So still we should think if there is anything we could find anything interesting using the dir commands. 
Let's have a look.

```python
>>> def foo():
... print "Meow"
...
>>> dir(foo)
['__call__', '__class__', '__closure__',
'__code__', '__defaults__', '__delattr__',
'__dict__', '__doc__', '__format__', '__get__',
'__getattribute__', '__globals__', '__hash__',
'__init__', '__module__', '__name__', '__new__',
'__reduce__', '__reduce_ex__', '__repr__',
'__setattr__', '__sizeof__', '__str__',
'__subclasshook__', 'func_closure', 'func_code',
'func_defaults', 'func_dict', 'func_doc',
'func_globals', 'func_name']
```

Oops! func_code.... another underlying bomb.
so lets dig deeper into this.
```python
>>> foo.func_code
<code object foo at 0x100509d30, file "<stdin>",
line 1>

>>> dir(foo.func_code)
['__class__', '__cmp__', '__delattr__', '__doc__',
'__eq__', '__format__', '__ge__',
'__getattribute__', '__gt__', '__hash__',
'__init__', '__le__', '__lt__', '__ne__',
'__new__', '__reduce__', '__reduce_ex__',
'__repr__', '__setattr__', '__sizeof__',
'__str__', '__subclasshook__', 'co_argcount',
'co_cellvars', 'co_code', 'co_consts',
'co_filename', 'co_firstlineno', 'co_flags',
'co_freevars', 'co_lnotab', 'co_name', 'co_names',
'co_nlocals', 'co_stacksize', 'co_varnames']
```
Let us have a look at the *co_code* attribute:
```python
>>> foo.func_code.co_code
'd\x01\x00GHd\x00\x00S'
```
surely we cannot change the hex that the python interpreter shows.
But let's try.
```python
>>> def foo():
... print "Meow"
...
>>> def evil_function():
... print "Kaboom!"
...
>>> foo()
Meow
```

Okay, so now,
```python
>>> def foo():
... print "Meow"
...
>>> def evil_function():
... print "Kaboom!"
...
>>> foo()
Meow
>>> foo.__setattr__("func_code",
 evil_function.func_code)
```
Let's try breaking:
```python
>>> def foo():
... print "Meow"
...
>>> def evil_function():
... print "Kaboom!"
...
>>> foo()
Meow
>>> foo.__setattr__("func_code",
 evil_function.func_code)
>>> foo()
Kaboom!
```

NOoooooooooooooooooooooooooooooooooooooooooo way! We did it again!.

####Idea redux: don’t expose dangerous implementation details

To get rid if this, we can use the previously used cpython library and the types.
So,
```python
from cpython import dictionary_of
from types import FunctionType
...
type_dict = dictionary_of(type)
del type_dict["__bases__"]
del type_dict["__subclasses__"]
function_dict = dictionary_of(FunctionType)
del function_dict["func_code"]
```
Let us recap our tactics:
    • Keyword blacklist (DO NOT USE)
    • Builtins whitelist
    • Import whitelist
    • Making important objects read-only (builtins)
    • Deleting problematic implementation details (__bases__, __subclasses__,func_code)
    • Deleting the ability to construct arbitrary code objects
###***We’ve implemented 80% of a full-fledged Python sandbox***

#####Let us once have a look at the entire code.
```python
builtins_whitelist = set((
 # exceptions
 'ArithmeticError', 'AssertionError', 'AttributeError', 'BufferError', 'BytesWarning', 'DeprecationWarning', 'EOFError',
 'EnvironmentError', 'Exception', 'FloatingPointError','FutureWarning', 'GeneratorExit', 'IOError', 'ImportError',
 'ImportWarning', 'IndentationError', 'IndexError', 'KeyError','LookupError', 'MemoryError', 'NameError', 'NotImplemented',
 'NotImplementedError', 'OSError', 'OverflowError','PendingDeprecationWarning', 'ReferenceError', 'RuntimeError',
 'RuntimeWarning', 'StandardError', 'StopIteration', 'SyntaxError', 'SyntaxWarning', 'SystemError', 'TabError', 'TypeError',
 'UnboundLocalError', 'UnicodeDecodeError', 'UnicodeEncodeError','UnicodeError', 'UnicodeTranslateError', 'UnicodeWarning',
 # constants
 'False', 'None', 'True', '__doc__', '__name__', '__package__', 'copyright', 'license', 'credits',
 # types
 'basestring', 'bytearray', 'bytes', 'complex', 'dict', 'float', 'frozenset', 'int', 'list', 'long', 'object', 'set', 'str',
 'tuple', 'unicode',
 # functions
 '__import__', 'abs', 'all', 'any', 'apply', 'bin', 'bool', 'buffer', 'callable', 'chr', 'classmethod', 'cmp', 'coerce',
 'compile', 'delattr', 'dir', 'divmod', 'enumerate', 'filter', 'format', 'getattr', 'globals', 'hasattr', 'hash', 'hex', 'id',
 'isinstance', 'issubclass', 'iter', 'len', 'locals', 'map', 'max', 'min', 'next', 'oct', 'ord', 'pow', 'print', 'property',
 'range', 'reduce', 'repr', 'reversed', 'round', 'setattr', 'slice', 'sorted', 'staticmethod', 'sum', 'super', 'type', 'unichr',
 'vars', 'xrange', 'zip',
 ))
 
def _safe_import(__import__, module_whitelist):
    def safe_import(module_name, globals={}, locals={}, fromlist=[], level=-1):
        if module_name in module_whitelist:
            return __import__(module_name, globals, locals, fromlist, level)
        else:
            raise ImportError("Blocked import of %s" % (module_name,))
 return safe_import
 
class ReadOnlyBuiltins(dict):
 def clear(self):
    ValueError("Read-only!")
    def __delitem__(self, key):
        ValueError("Read-only!")
    def pop(self, key, default=None):
        ValueError("Read-only!")
    def popitem(self):
        ValueError("Read-only!")
    def setdefault(self, key, value):
        ValueError("Read-only!")
    def __setitem__(self, key, value):
        ValueError("Read-only!")
    def update(self, dict, **kw):
        ValueError("Read-only!")
    class Sandbox(object):
        def __init__(self):

    import sys
    from types import FunctionType
    from cpython import dictionary_of
    original_builtins = sys.modules["__main__"].__dict__["__builtins__"].__dict__
    for builtin in original_builtins.keys():
        if builtin not in builtins_whitelist:
            del sys.modules["__main__"].__dict__["__builtins__"].__dict__[builtin]
    original_builtins["__import__"] = _safe_import(__import__, ["string", "re"])
    safe_builtins = ReadOnlyBuiltins(original_builtins)
    sys.modules["__main__"].__dict__["__builtins__"] = safe_builtins
    type_dict = dictionary_of(type)
    del type_dict["__bases__"]
    del type_dict["__subclasses__"]
    function_dict = dictionary_of(FunctionType)
    del function_dict["func_code"]
    def execute(self, code_string):
        exec code_string
```

Alas! at last.
###Experiments
• How does an alternative Python implementation like PyPy handle these issues?
• How does the CPython interpreter compile and run bytecode?
• What does the Python stack look like?
• How do ctypes work?
• How can the operating system help provide a secure environment?

Thankyou for taking the time to read this. :-)
