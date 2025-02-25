<!--
  © 2021-2022 Intel Corporation
  SPDX-License-Identifier: MPL-2.0
-->

# A short hacking guide to dmlc

The DML compiler (`dmlc`) is a prototype that has expanded far beyond
its comfort zone. It works (kind of), but it suffers from lack of
clear design and lots of historical baggage. This text is an attempt to
make it easier for you to find your way around the sources and
understand what is going on inside the compiler.

## dml/dmlc.py

The "main program" is the dmlc.py file. It handles the command-line
arguments and tells the rest of the compiler which files to read and
what output to create. I think it should be pretty straightforward to
understand.

It has a couple of hidden utilities for you, the developer. If you set
the DMLC_DEBUG environment variable, it will print a backtrace when it
gets an internal compiler error, instead of saving it to a file and
just giving you the file name. This helps quite a lot when testing
your changes.

There are traces of various profiling attempts. The simplest is by
setting time_dmlc to True, which will print a little information about
how much time the compiler stages take, although this isn't all that useful.

You can also set the DMLC_PROFILE environment variable to create a
cProfile profile.

There is a similar environment variable DMLC_HEAPY to do some
profiling with heapy/guppy, but that's a bit more complicated.

## dml/globals.py

The globals module is just a place where some information used
throughout the compiler is thrown. For instance, it has a variable
dml_version that contains the version (major,minor) of the DML device
being compiled (note that this can be different from the DML version
of the source file containing the statement it is currently
processing.

## dml/objects.py

This file defines the DMLObject base class and subclasses for every
kind of DML "object" that can appear in a device, like Device, Group,
Register, Connection etc. Mostly, these are just struct-like
datatypes, but there is some common functionality in the base class
and a little in other classes, especially Method.

## dml/dmlparse.py, dmllex.py

The parsers are generated by PLY. Which means that they are rather
slow. The DML reference manual contains a grammar that is generated
automatically from this grammar.

## dml/ast.py

The abstract syntax tree (AST) generated by the parser is a mix of an
older format and a newer format. Originally, it produced tagged tuples
like `('assign', ('var', "foo"), ('int', 17))`. Some time along the way
it was changed so that the AST nodes are not tuples but objects of a
subclass of ast.AST, at least sometimes. These objects have provisions
for accessing them as if they were the old-style tuples, so `node[0]` is
the same as `node.kind`.

## dml/structure.py

This is an attempt to put the preliminary stages of the compiler in a
separate module, but it is a bit random. The functions in this module
are called from the `process()` function in `dmlc.py`, and they are

* `mkglobals()` which goes through the AST and looks for global names and adds
  them to the global scope. This includes things declared with "constant" and
  external symbols, but also typedefs.

* `check_types()` that performs checks on typedefs to see that they
  don't look broken in some way.

* `mkobj()` which does the major work of going through the AST and creating the
  DMLObject tree. It will merge AST information for nodes with the same name
  and creates all parameter values, and provides values for the parameters
  marked with "auto" in the sources.

It also has some utility functions for managing register maps and
methods.

## dml/ctree.py

The ctree module has lots of classes for representing code. It is a
bit complicated and not very cleanly designed.

The classes mostly corresponds to the kind of code that can be written
in DML. Some are special and synthesised or generated by the compiler
code.

The two main categories of classes are statements and expressions,
with lots of subclasses each. Statement objects have a `toc()` method
that is used to generate actual C code using the `output.out()`
function.

Expression objects have a number of common properties. Most of them are
documented in the base class. The `ctype()` method is used to return a
type object, if one is known.

The classes are not instantiated directly, instead factory functions
are used. For instance, to create an Add object, the mkAdd function is
called. These factory functions often just create an object, but in
some cases it tries to do simplifications and may return an object of
another type. For instance `mkAdd(x, mkInt(0))` will return `x`.

## dml/codegen.py

This file does the conversion of an objects.Method object to a ctree
representation of the code. It does inlining and can do partial
evaluation by specializing methods with untyped parameters. The code
that tries to figure out which methods to actually generate C
functions for is a bit hairy.

## dml/c_backend.py

The C backend goes through the object tree as created by the structure
module and finds all interfaces, attributes etc and uses codegen to
generate code for them, and finally it puts everything in a bunch of C
files.

## dml/output.py

This module has a function `out()` that is used to write stuff to the
generated source. It has built-in support for tracking indentation and
line numbers to generate well-indented code and proper #line
directives. It also has some functions for juggling multiple output
files.
