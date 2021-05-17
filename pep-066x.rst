PEP: 66x
Title: Memory mappable .pyc file format
Author: Mark Shannon <mark@hotpy.org>
Status: Active
Type: Standards Track
Content-Type: text/x-rst
Created: xx-May-2021
Post-History: xx-May-2021

Abstract
========

This PEP proposes that .pyc files should use a format that allows them to be loaded into 
memory and executed from without any decoding stage or mutation. 
This will allow faster loading of the code objects for a module, and should speed up loading time of modules considerably.

Currently, loading a code object requires an unmarshalling phase. All names and constants must be unmarshalled,
even if they are never used.
In particular, code that is never run or only run in case of errors will load much faster as no preparatory 
unmarshalling phase will be required.


Motivation
==========

[Clearly explain why the existing language specification is inadequate to address the problem that the PEP solves.]


Rationale
=========

[Describe why particular design decisions were made.]


Specification
=============

The exact format of the pyc file is not part of this PEP as it is expected to evolve over time.
However, we will describe the design and layout in broad terms.

The pyc will consist of the following sections:

1. Header
2. Code
3. Constant objects
4. Constant strings
5. Binary data
6. Meta-data

The header will always start with eight bytes::

* The first four bytes are the ascii string ".pyc".
* The next two bytes are the format version number.
* The last two bytes are the instruction set version.

Necessary changes to the bytecode instruction set
'''''''''''''''''''''''''''''''''''''''''''''''''

Python bytecode requires references to constant objects, especially strings.
These objects need to be created at runtime, so the pyc file must include instructions for creating
these objects. Prior to this PEP that was done by unmarshalling the code object; 
the .pyc file was simply a short header followed by the marshalled form of the code object.

Since the marshalling phase is no longer present, constants and strings need to created by the bytecode interpreter.

To support this, ``LOAD_CONSTANT`` will be replaced by ``LAZY_LOAD_CONSTANT`` and a number of maker
instructions will be added:

* ``LAZY_LOAD_CONSTANT`` -- If the value in constant slot is not NULL, push it.
  Otherwise jump to instruction sequence for that constant.
* ``MAKE_STRING`` -- Creates a string from binary data
* ``MAKE_INT`` -- Loads a small integer
* ``MAKE_LONG`` -- Creates an integer (of arbitrary size) from binary data and adds it to the stack.
* ``MAKE_FLOAT`` -- Creates a float from binary data
* ``MAKE_COMPLEX`` -- Creates a complex number from the two floats on top of the stack.
* ``MAKE_CODE_OBJECT`` -- Makes a code object.
* ``LOAD_COMMON_CONSTANT`` -- Loads one of a fixed set of common constants, like ``None``, ``()``, ``AssertionError``, etc.
* ``RETURN_CONSTANT`` -- Sets the nth constant to the value TOS and returns to the instruction after ``LAZY_LOAD_CONSTANT``

All instructions that need a name, will lazily create the name on demand using ``MAKE_STRING``.

All the above bytecodes are normal bytecodes and can used in any code, with the exception of ``RETURN_CONSTANT`` which can only be used
to terminate the code for ``LAZY_LOAD_CONSTANT``.

Example
-------

The bytecode for the function::

  def f():
    return (1, "Hi")

is, in 3.10::

  LOAD_CONSTANT 0
  RETURN_VALUE

With this PEP, it would become::

  LAZY_LOAD_CONSTANT 0
  RETURN_VALUE

And the constant object description at index ``0`` would contain the following bytecode::

  MAKE_INT 1
  LAZY_LOAD_CONSTANT 1
  BUILD_TUPLE 2
  RETURN_CONSTANT 0

The constant object description at index ``1`` would contain the following bytecode::

  MAKE_STRING 0
  RETURN_CONSTANT 1

The string description at index ``0`` would contain the offset of the binary data for "Hi".

While this seems a lot more complex, the unmarshalling process was doing the same work, but less effciently and eagerly.
It also gives the compiler some flexibility.
Consider code that uses the the constant ``(1, "Hi")`` only once. In that case there is no need to make it a constant and
instead of ``LOAD_CONSTANT 0``, the code to create the object can be inlined::

  MAKE_INT 1
  MAKE_STRING 0
  BUILD_TUPLE 2

Version 0
'''''''''

We describe a "version 0" of the format, to illustrate one possible design.
Unless otherwise specified:

* All numbers are in little endian format
* Variable sized integers are encoded using varint128, signed numbers pre-encoded as ``(abs(n) << 1) | (n < 0)``.
* All metadata offsets are from the start of the metadata section.
* All other offsets are from the start of the ".pyc" file.
* All fields have an alignment that is at least their size.

Header
''''''

Contains::

  ".pyc"
  version: u2
  n_code: u2
  meta_start: u4
  total_size: u4

The ``meta_start`` field is the offset to the start of the metadata section,
so it can loaded independently from the rest of the .pyc file.

Code section
''''''''''''

``n_code`` 4 byte entries, each specifying the offset of the code object data.

Each code data contains::

  co_flags : u4
  co_argcount: u4
  co_posonlyargcount: u4
  co_kwonlyargcount: u4
  co_nlocals: u4
  co_stacksize: u4
  co_name: u4 (index into string table)
  co_exceptiontable: u4 (binary data offset)
  co_filename: u4 (meta data offset for string)
  co_locationtable: u4 (meta data offset)
  co_docstring: u4 (meta data offset for string)
  co_codelength: u4
  co_code: u2 * co_codelength
  co_nvars: u4
  co_varnames: u4 * co_nvars (indexes into string table)

Constant objects
''''''''''''''''

Object table::

  n_object: u4
  object_code: u4 * n_object

Starts with a 4 bytes integer containing the number of constants, followed by a four byte integer per object.
Each four byte entry describes the offset into the object code which creates the constant.

Object code::

  codelength: u4
  code: u2 * codelength

Constant strings
''''''''''''''''

String table::

  n_strings: u4
  string_offset: u4 * n_strings

The offset is into the binary data containing the variable sized length, followed by the utf8 encoded text.

Binary data
'''''''''''

Contains all the data required to create objects, including strings.
In has no structure. The meaning of the bytes within this section is determined by what is indexing it.

Meta data
'''''''''

Contains all the meta data, such as line numbers.

Format::

  n_files: u4
  files: u4 * n_files (offsets to filenames in binary_data)
  location_table
  binary_data

Location table
--------------

The line table is organized into sections::

  line_sections: u4

Each section is exactly 64 bytes long, except the last section which may be shorter.
Each section is organized as follows::

  line_number: u4
  instruction_offset: u4
  filename_index: u1
  entries: u1 

There is one entry per instruction in the section, varint128 encoded::
  
  * line delta (signed)
  * column offset

The first entry of the first section holds the location of the definition, so the Nth entry 
holds the location of the (N-1)th instruction.

Binary data
-----------

Contains all the data for metadata strings.
In has no structure. The meaning of the bytes within this section is determined by what is indexing it.

Runtime objects
'''''''''''''''

The code object will need to hold a pointer to the pyc file, and and offset to the start of the
data for that code object.

In addition it will need areference to the shared array of constants and names::

  struct _code_object {
      PyObject_HEAD
      char *pyc;
      int offset;
      PyObject *consts_and_names; /* Strong reference to shared array of names and constants */
      PyObject **names; /* == &consts_and_names->items[n_consts] */
  };

For efficiency, names are constants are stored in a common array,
so that ``const_n == names[-1-n]`` and ``name_n == names[n]``.

Backwards Compatibility
=======================

The new pyc files will be completely incompatible with old format,
but pyc files are not compatible across versions anyway.

There will be no change to the language.
The code object changes from version to version. This PEP will require
larger than usual changes to the code object, but gratuitious breakage will be avoided. 

Security Implications
=====================

[How could a malicious user take advantage of this new feature?]



Reference Implementation
========================

[Link to any existing implementation and details about its state, e.g. proof-of-concept.]


Rejected Ideas
==============

[Why certain ideas that were brought while discussing this PEP were not ultimately pursued.]


Open Issues
===========

[Any points that are still being decided/discussed.]


References
==========

[A collection of URLs used as references through the PEP.]


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.



..
    Local Variables:
    mode: indented-text
    indent-tabs-mode: nil
    sentence-end-double-space: t
    fill-column: 70
    coding: utf-8
    End:

