# PEP 7 -- Style Guide for C Code

| PEP:              | 7                                                            |
| ----------------- | ------------------------------------------------------------ |
| **Title:**        | Style Guide for C Code                                       |
| **Author:**       | Guido van Rossum <guido at python.org>, Barry Warsaw <barry at python.org> |
| **Status:**       | Active                                                       |
| **Type:**         | Process                                                      |
| **Created:**      | 05-Jul-2001                                                  |
| **Post-History:** |                                                              |

Contents

- [Introduction](#Introduction)
- [C dialect](#c-dialect)
- [Code lay-out](#code-lay-out)
- [Naming conventions](#naming-conventions)
- [Documentation Strings](#documentation-strings)
- [References](#references)
- [Copyright](#copyright)

# Introduction

This document gives coding conventions for the C code comprising the C implementation of Python. Please see the companion informational PEP describing style guidelines for Python code [[1\]](https://www.python.org/dev/peps/pep-0007/#id2).

Note, rules are there to be broken. Two good reasons to break a particular rule:

1. When applying the rule would make the code less readable, even for someone who is used to reading code that follows the rules.
2. To be consistent with surrounding code that also breaks it (maybe for historic reasons) -- although this is also an opportunity to clean up someone else's mess (in true XP style).

# C dialect

- Python versions before 3.6 use ANSI/ISO standard C (the 1989 version of the standard). This means (amongst many other things) that all declarations must be at the top of a block (not necessarily at the top of function).

- Python versions greater than or equal to 3.6 use C89 with several select C99 features:

  - Standard integer types in `<stdint.h>` and `<inttypes.h>`. We require the fixed width integer types.
  - `static inline` functions
  - designated initializers (especially nice for type declarations)
  - intermingled declarations
  - booleans
  - C++-style line comments

  Future C99 features may be added to this list in the future depending on compiler support (mostly significantly MSVC).

- Don't use GCC extensions (e.g. don't write multi-line strings without trailing backslashes).

- All function declarations and definitions must use full prototypes (i.e. specify the types of all arguments).

- Only use C++ style // one-line comments in Python 3.6 or later.

- No compiler warnings with major compilers (gcc, VC++, a few others).

# Code lay-out

- Use 4-space indents and no tabs at all.
- No line should be longer than 79 characters. If this and the previous rule together don't give you enough room to code, your code is too complicated -- consider using subroutines.
- No line should end in whitespace. If you think you need significant trailing whitespace, think again -- somebody's editor might delete it as a matter of routine.
- Function definition style: function name in column 1, outermost curly braces in column 1, blank line after local variable declarations.

```C
static int
extra_ivars(PyTypeObject *type, PyTypeObject *base)
{
    int t_size = PyType_BASICSIZE(type);
    int b_size = PyType_BASICSIZE(base);

    assert(t_size >= b_size); /* type smaller than base! */
    ...
    return 1;
}
```

Code structure: one space between keywords like `if`, `for` and the following left paren; no spaces inside the paren; braces are required everywhere, even where C permits them to be omitted, but do not add them to code you are not otherwise modifying. All new C code requires braces. Braces should be formatted as shown:

```C
if (mro != NULL) {
    ...
}
else {
    ...
}
```

The return statement should *not* get redundant parentheses:

```C
return albatross; /* correct */
return(albatross); /* incorrect */
```

- Function and macro call style: `foo(a, b, c)` -- no space before the open paren, no spaces inside the parens, no spaces before commas, one space after each comma.
- Always put spaces around assignment, Boolean and comparison operators. In expressions using a lot of operators, add spaces around the outermost (lowest-priority) operators.
- Breaking long lines: if you can, break after commas in the outermost argument list. Always indent continuation lines appropriately, e.g.:

```C
PyErr_Format(PyExc_TypeError,
             "cannot create '%.100s' instances",
             type->tp_name);
```

When you break a long expression at a binary operator, the operator goes at the end of the previous line, and braces should be formatted as shown. E.g.:

```C
if (type->tp_dictoffset != 0 && base->tp_dictoffset == 0 &&
    type->tp_dictoffset == b_size &&
    (size_t)t_size == b_size + sizeof(PyObject *))
{
    return 0; /* "Forgive" adding a __dict__ only */
}
```

- Put blank lines around functions, structure definitions, and major sections inside functions.

- Comments go before the code they describe.

- All functions and global variables should be declared static unless they are to be part of a published interface

- For external functions and variables, we always have a declaration in an appropriate header file in the "Include" directory, which uses the `PyAPI_FUNC()` macro, like this:

  ```C
  PyAPI_FUNC(PyObject *) PyObject_Repr(PyObject *);
  ```

# Naming conventions

- Use a `Py` prefix for public functions; never for static functions. The `Py_` prefix is reserved for global service routines like `Py_FatalError`; specific groups of routines (e.g. specific object type APIs) use a longer prefix, e.g. `PyString_` for string functions.
- Public functions and variables use MixedCase with underscores, like this: `PyObject_GetAttr`, `Py_BuildValue`, `PyExc_TypeError`.
- Occasionally an "internal" function has to be visible to the loader; we use the `_Py` prefix for this, e.g.: `_PyObject_Dump`.
- Macros should have a MixedCase prefix and then use upper case, for example: `PyString_AS_STRING`, `Py_PRINT_RAW`.

# Documentation Strings

- Use the `PyDoc_STR()` or `PyDoc_STRVAR()` macro for docstrings to support building Python without docstrings (`./configure --without-doc-strings`).

  For C code that needs to support versions of Python older than 2.3, you can include this after including `Python.h`:

  ```c
  #ifndef PyDoc_STR
  #define PyDoc_VAR(name)         static char name[]
  #define PyDoc_STR(str)          (str)
  #define PyDoc_STRVAR(name, str) PyDoc_VAR(name) = PyDoc_STR(str)
  #endif
  ```

- The first line of each function docstring should be a "signature line" that gives a brief synopsis of the arguments and return value. For example:

  ```C
  PyDoc_STRVAR(myfunction__doc__,
  "myfunction(name, value) -> bool\n\n\
  Determine whether name and value make a valid pair.");
  ```

  Always include a blank line between the signature line and the text of the description.

  If the return value for the function is always None (because there is no meaningful return value), do not include the indication of the return type.

- When writing multi-line docstrings, be sure to always use backslash continuations, as in the example above, or string literal concatenation:

  ```C
  PyDoc_STRVAR(myfunction__doc__,
  "myfunction(name, value) -> bool\n\n"
  "Determine whether name and value make a valid pair.");
  ```

  Though some C compilers accept string literals without either:

  ```C
  /* BAD -- don't do this! */
  PyDoc_STRVAR(myfunction__doc__,
  "myfunction(name, value) -> bool\n\n
  Determine whether name and value make a valid pair.");
  ```

  not all do; the MSVC compiler is known to complain about this.

# References

| [[1\]](https://www.python.org/dev/peps/pep-0007/#id1) | [PEP 8](https://www.python.org/dev/peps/pep-0008), "Style Guide for Python Code", van Rossum, Warsaw (http://www.python.org/dev/peps/pep-0008) |
| ----------------------------------------------------- | ------------------------------------------------------------ |

# Copyright

This document has been placed in the public domain.

Source: https://github.com/python/peps/blob/master/pep-0007.txt

