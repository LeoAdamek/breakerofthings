---
layout: post
title: LISPeration
date: "2025-04-01"
---

Introduction
============

Recently I've been thinking about early computing, and early programming.
Primarily, this has involved looking further into high-level programming
languages predating the arrival of C, and with it, UNIX.

The influence of the C programming language on modern computing is almost
impossible to overstate. Every major operating system is written primarily, if
not entirely, in C. Nearly all software is written in C, a language which has a
compiler or runtime written in C, or a language which has a lineage traceable
back to C. 

I think computing today would look _very_ different without C. Much like the
UNIX epoch of 1st January 1970, the inventions of the C programming language,
and the UNIX operating system themselves represent an epoch in computing.
Computing itself can largely be thought of as _Before C_ and _After C_.

So in that thought I decided to look at programming languages, not including
machine-specific assembler dialects, that predate C and UNIX.

In particular though, the very early programming languages, which had original
releases in or before 1960. Especially of not are, FORTRAN, COBOL, ALGOL60, and
of course, LISP.

Of these, LISP stands out as perhaps the only one of these languages which, even
in its earliest versions as released before 1960, is still a language a modern
programmer could find themselves using and finding enjoyment in using.

Enter LISP
==========

For the purposes of this post, we'll be talking about _Common LISP_ which is the
most direct descendant of John McCarthy's original 1956 LISP language.

LISP development near enough always involves a REPL. An interactive process
which allows the user to enter LISP expressions (known as _Symbolic
Expressions_, or _sexp_ for short), and prints the result. In fact, a full
application can be built entirely within this interactive environment.

Using this interactive loop, the programmer can build up their program, defining
variables, functions, and other language primitives. When they're happy with the
program the entire environment, as it exists can be _imaged_, and stored on disk
to be reloaded and rerun. To modify the program, one simply has to start the
program up again, and begin modifying its contents.
Functions, variables, and other values can be changed and replaced in a program
as it's running.

This isn't the *only* way to write LISP programs, it's perfectly valid to
write out your whole program in one or more files and then create an image from
these sources, and this is how most LISP programs are written, along with modern
dependency management tools like _Quicklisp_, and build tools like _ASDF_.

LISP In No Time At All
======================

The most obvious trait of the LISP language, is undoubtedly the parentheses.
At first the parentheses may seem intimidating, and hard to read. I think, once
you start to read it, it actually becomes incredibly readable and logical.

Another thing to note is that identifiers in LISP are _case-insensitive_, and
by convention, the compiler will default to ouputting them in UPPERCASE.

LISP code is composed of _symbolic expressions_, each of these expressions is
enclosed by parentheses. Due to this, the structure of the syntax is made clear
to the programmer, leaving no ambiguity.

Generally a symbolic expression starts with a function name, followed by a list
of its arguments, it can be expressed as:

    (<FUNCTION-NAME> [<ARGUMENT> ])

Pretty much everything is a function, including arithmetic operations. This
fact, combined with the syntax, provides a couple of interesting capabilities:

* With subtraction not being an _infix_ operation, leaves the hyphen character
  available for use within names. It is convention within LISP to use names
  which are separated by hyphens, also known as "kebab case" as the names appear
  to have a skewer going through them.
* Arithmetic operations are now _varadic_, they can accept any number of
  arguments[^1] and will operate across them. 
  This means that instead of `+` being a simple addition operator of two values, 
  it's the _summation_ operation Σ. For example `(+ 1 2 3 4)` gives the answer `10`.
  Similarly, `*` becomes the _product_ function, Π.

[^1]: The total number of arguments in a LISP function call is limited by the
implementation. The standard requires the implementation export a constant
called `lambda-paramters-limit` which is required to be at least 50.
SBCL, the most popular open source LISP implementation, has a system-specific
limit which can vary. I've seen values from 2³⁰, already a high value, to 2⁶²!

Basic Common LISP code consists of:

* variables
* functions
* macros
* parameters

Variables are named places to hold a value. They're defined using `defvar` and
are generally done at the top-level, to define a value which will be retained
all the time the program is running.

Functions are pretty self-explanatory, they work much like functions in any
other language whereby they are groups of expressions which can optionally take
input values, and can optionally return one, and in LISP, multiple, output
values. These are defined using `defun`.

Macros are similar to functions, but are evaluated at _compile-time_ and can
include not only the code to be evaluated at compile-time, but also can generate
new code which is then in turn compiled. These are defined using `defmacro` and
several core language features such as `defun` are actually implemented using
macros.

Parameters are perhaps the most unique of these four basic definitions. As
macros are to functions, parameters are the _inverse_ of this to variables.
A parameter defines a name for a value, but the value of this variable is
re-evaluated every time it is referenced, rather than when it is defined.

Take the following example:

~~~lisp 
(defvar variable-value (get-universal-time))
(defparameter parameter-value (get-universal-time))
~~~

In this example, the variable `variable-value` will have the value of
`(get-universal-time)` when the `defvar` expression is evaluated, and this value
will not change unless it is explicitly changed using a set expression.
The parameter `parameter-value` however, will be defined, but it doesn't have a
stored value, its value will be evaluated when it is referenced, and will be
reevaluated each time it is referenced.

You can think of the difference between `defvar` and `defparamter` as being
similar to the following JavaScript:

~~~javascript
const variable_value = Date.now();
const parameter_value = () => Date.now();
~~~

The major difference being that in LISP, a parameter can be used and referenced
just like a variable, and doesn't need any function-call syntax.


There are other things you can `def` in common LISP, such as structures, as well
as an object-oriented model called _Common LISP Object System (CLOS)_ which
allows for object-oriented programming within Common LISP.

LISP Firsts
===========

LISP is home to a number of features for which it was first to implement, or was
the first widely successful language to implement, the use of which can still be
widely seen today.

Lambda/Anonymous Functions
--------------------------

LISP supports anonymous lambda functions, including inline functions.

For example, if we wanted to iterate over a list with a quick-and-simple
function to raise 2 to the power of that number, we could write:

~~~lisp
(dolist (lambda (n) (expt 2 n)) my-list-of-numbers)
~~~

This is something that other languages of the time simply could not do, a
function definition had to be at the top-level of the program, and to iterate
over a list and apply this operation would require an externalised definition of
that function.

The term Lambda-function, while derived from mathematics, is still widely in use
today. Most notably in Haskell (where the `\` character is used as a stand-in
for λ), and Python where anonymous functions are declared using `lambda`.

Documentation Strings (Docstrings)
---------------------------------

LISP supports inline, online, documentation of functions using a _docstring_.
After defining the list of arguments for a function, if the next _form_ in the
function is a string, it is treated as being documentation for the function.

~~~lisp 
(defun rectangle-perimiter (width height)
    "Calculates the perimeter of a rectangle of `width' and `height'"
    (+ (* 2 width) (*2 height)))
~~~

You can then use the `describe` or `doc` functions in your LISP interactions to
be shown the this message, used to describe the function.

This is another language feature also very much present in Python.

Optional Arguments and Keyword Arguments
----------------------------------------

LISP supports both optional arguments, and keyword arguments, which is
especially useful when you have many arguments, only a few of which are actually
to be used. It also supports _varadic functions_, functions which take an
arbitrary number of arguments.

When defining a function you supply a list of arguments, within this list
special keywords, prefixed with `&`, can be used to specify properties for the
remaining arguments in the list.

Any arguments after `&optional` are optional, their value will default to `nil`
unless a different default is specified, or a value is supplied by the caller.

Any arguments after `&key` are _keyword arguments_, the caller must give an even
number of arguments, first specifying a _keyword_, prefixed with a `:`, to
denote which argument is being given and then the value being given for that
argument.

`&rest` can appear only before the last argument in the list, when given the
last argument will contain a list of all remaining arguments.

For example:

~~~lisp
(defun my-special-function (a b &optional c d &key f g &rest r)
    (...))

(my-special-function required-a required-b 
                     optional-c 
                     :f value-f 
                     :g value-g
                     values will be combined into list-R
)
~~~

In the above definition:

* the arguments `A` and `B` are required. 
* `C` and `D` are optional, but positional. 
  To specify `D`, you must first specify `C`.
* `F` and `G` are optional keywords, to specify them you 
  first specify the keywords `:F` or `:G`, followed by the value.
* `R` is a _rest_ argument, all arguments after `G` will be put into a list
  which will be passed into the argument `R`.

Killer Features of LISP
=======================

There's a few features of LISP I really want to highlight as being especially
useful, powerful, or just interesting. 

`let` Expressions
-----------------

LISP's `let` expression is a macro which allows a list of values to be named and
set within a single scope. Here's an example:

~~~~lisp
(let ((name "World")
      (time (get-universal-time)))
     (format t "Hello ~a, the time is ~a" name time))
~~~~

Within the `let` expression, the name `name` has the value `"World"` and the
name `time` has a value equal to the result of the sub-expression `(get-universal-time)`.

`get-universal-time` is a function which gets the current atomic time, returned
as a number of seconds since 1st January **1900**, so is equal to the current
UNIX time, plus 70 years.

`let` expressions, as well as a few other similar macros, can beused to create
very powerful and easy to understand code because the scope of a variable's
existance can be made both explicit and direct. The variable exists only within
the `let` expression in which it was defined.


The Type System
---------------

LISP is fundamentally a dynamically typed language, by default function
parameters and return values have no set type. A type is only enforced by how
the values are used. 

The `declare` expression allows the programmer to make explicit declarations to
the compiler. The most obvious of which is to explicitly declare types for
function parameters and return values for functions.

A simple declaration would be something like this:

~~~lisp
(declare (type string X Y))
~~~

This declares that the values of `X` and `Y` are strings.
However, the LISP type system goes beyond this. For example if we want to
specify that a value not only must be numeric, but must fall within a specific
_range_? For example, a function that converts degrees to radians which only
accepts numbers from zero to 360?

~~~lisp
(defun degrees-to-radians (degrees)
    (declare (type (real 0 360) degrees))
    (* 2 pi (/ degrees 360)))
~~~

This declaration specifies not only that `degrees` must be a real number, but
that it must fall within the range of 0 to 360 inclusive. Attempting to call
this function with non-number value, a complex number value, or a value outside
this range will cause an error.

You can define your own named types in LISP using the `deftype` macro. A type
can be any combination of the built-in primitives, combined with built-in
operations such as `or` and `and`. For example:

~~~lisp
(deftype symbol-or-string () (or symbol string))
~~~

Declares that the type `symbol-or-string` is the type which is either a symbol
or a string.

Or you can use the `satisfies` macro to define a test function which performs
whatever checks you like and returns if the value is a match, or not.

Expression-Level Compilation Settings
-------------------------------------

There are other things you can do with `declare` expressions, such as tuning the
compiler via `optimize`, which allows the programmer to indicate to the compiler
how it should prioritise on a scale of 0 to 3, zero being no consideration made
and 3 being highest priority, the following attributes of the compilation:

* speed of the compilation processes itself `compilation-speed`
* the execution speed of the compiled code  `speed`
* the size and memory usage of the compiled code `space`
* ease of debugging `debug`
* runtime safety. `safety`

Implementations can define additional optimization priorities, but these five
are defined by the Common LISP standard, so should be _accepted_, even if
ignored, by any implementation.

Any combination of these properties can be specified at a function-level
allowing individual functions to be compiled with different settings. This way a
well-tested function that is performance critical can be compiled to focus
entirely on runtime performance, while other parts of the program can be
compiled with a focus on runtime safety and debugging capabilities.

For example:

~~~lisp
(defun factorial (n)
    (declare (type (integer 1 *) n) 
             (values (integer 1 *))
             (optimize (speed 3)
                       (compilation-speed 0)
                       (debug 0)
                       (safety 1)))
    (if (< n 2)
        n
        (* n (factorial (- n 1)))))
~~~

This `FACTORIAL` function specifies that maximum priority be given to the
execution speed of the compiled code, minimal consideration made for runtime
safety, and no consideration is to be made for debugging or speed of
compilation.

We've also declared that the input argument, `N`, must be an integer in the
range of 1-*, where * indicates no upper limit.
Similarly we've declared the result of this function will be a single value,
which is also an integer the same range.

HyperSpec
---------

While every LISP implementation has its own extensions, features and
capabilities, they all implement the Common LISP standard, formally known by
the rather opaque name _x3j13_.

Within this standard are a total of 978 defined _symbols_, which encompass
everything that is defined within a standards-compliant Common LISP environment.
This includes global parameters, constants, functions, macros, and "special"
definitions. 

In LISP a symbol can name multiple things simultaneously. A symbol can name
one item from each of the following groups at the same time:

* A function or macro (_function-specifier_)
* A variable or parameter. (_value-specifier_)
* A type, structure, or class. (_type-specifier_)

This would normally be an _anti-pattern_ and something to be avoided. However,
within LISP, the context as to _which_ of these is being referred is made clear
by the syntax and structure of the language.

In LISP, a function can appear in only two places:
 
1. As a _function call_, at the very start of an expression, in which case it will
   be preceded immediately by an open parenthesis as in `(FUNCTION-NAME`.
2. As a _function reference_ where the function is being used as a value, which
   has a prefix of `#'`, as in `#'FUNCTION-NAME`. Without the `#'` prefix, the
   symbol would be treated as a _value-specifier_.

Similarly the name for a type, structure, or class, can only appear in specific
places that call for a type-name, such as in a `(declare (type))` or another
type definition.

A complete list of which can be found [here](https://www.lispworks.com/documentation/HyperSpec/Front/X_AllSym.htm)

Image-Based Programming
-----------------------

This one is a bit more complicated, and not necessarily always true. One thing
many LISP implementations have is the ability to perform _image-based
compilation_. 

Instead of reading in source code from disk, passing it through a compilation
pipeline and producing an executable program of that source, LISP compilers can
often create an executable binary out of a running image. In this mode, not only
are all LISP sources compiled, but variables with their current values as well
as the LISP language itself are compiled into a single executable. This is much
more like taking a snapshot of a virtual machine, than just compiling a program.

If your LISP program has a bug, instead of needing to replicate all the
conditions to cause the bug, which may not be known, you can instead compile an
image of the program _in the state where the bug is presenting_, and then use
the REPL to inspect the program in this state and devise a fix.

Of course, this facility has the potential for misuse, or to create programs
which can exhibit unexpected behaviour, as their execution is dependant on the
contents of the image. It also creates binaries that are either rather large, or
require compression, due to the inclusion of the full stack, heap values, and
LISP environment itself with the compiler baked in.


Should We Be Writing More LISP?
===============================

These days (Common) LISP has fallen into the realms of being a niche
language. It's seen as being somehow antithetical to modern software
engineering, or the s-expression syntax puts people off... Whatever the reasons,
Common LISP doesn't get the consideration in modern software development that I
believe it should.

Common LISP may not be suitable for _every_ problem, but it can do a reasonably
good job of most problems. With _Quicklisp_ there exists a wealth of great
libraries for all sorts of purposes. Tools for building Web APIs or full-stack
Web applications, tools for GUI applications, databases, and so on.

LISP's ability to get instant feedback about your program, inspect its
operation, modify its behaviour and to compose values and behaviours into a full
piece of software creates a development experience that is above all-else,
_fun_.
