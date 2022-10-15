---
title: Why Are Python Integers 28 Bytes Wide?
date: 2022-10-10T13:28:57-04:00
categories: ["python", "c", "low-level", "internals"]
slug: sizeof-python-integers
toc: true
published: true
---

The title isn't 100% accurate, and most of this stuff depends on the machine
where python is run. My computer uses a 64-bit architecture.

## Integers are suprisingly large

Whats the deal with integers in Python?

```python
Python 3.9.12 (main, Mar 26 2022, 15:52:10)
[Clang 13.0.0 (clang-1300.0.29.30)] on darwin
Type "help", "copyright", "credits" or "license" for more information
>>> from sys import getsizeof
>>> getsizeof(0)
24
>>> getsizeof(1)
28
>>> getsizeof(2)
28
...
>>> getsizeof(2 ** 30)
32
```

What about larger numbers? Maybe the first `100` powers of `2`?

```python
>>> getsizeof(2 ** 29), getsizeof(2 ** 30)
(28, 32)
>>> size_to_idx = {}
>>> for i in range(100):
...     size = getsizeof(2 ** i)
...     if size not in size_to_idx:
...             size_to_idx[size] = i
...
>>> {v: k for k, v in size_to_idx.items()}
{0: 28, 30: 32, 60: 36, 90: 40}
```

The above tells us that the sequence looks something like:

Number       | Size (bytes)
------------:|:------------
 `0`         | 24
 `2**0`      | 28
 `2**30`     | 32
 `2**60`     | 36
 `2**90`     | 40
 

We'd expect `2 ** 120` to take up `44` bytes if that's correct.

```python3
>>> getsizeof(2 ** 120 - 1), getsizeof(2 ** 120)
(40, 44)
```

Bingo!

We still don't know how this sequence comes about. What is going on under the hood?

[header-size]: https://github.com/python/cpython/blob/da6c78584b1f45ce3766bf7f27fb033169715292/Include/internal/pycore_object.h#L222

## Meet: `PyObject` & family

All data in Python has the same base type: [`PyObject`][pyobj]. Things
such as `int`, `class` etc are all represented by a similar blob under the hood.
The type used to represent numbers in Python is called [`PyLongObject`][pylong]

To understand why 0 can occupy `24` bytes is a good an excuse to dive into
Python source code.

[pyobj]: https://docs.python.org/3/c-api/structures.html#c.PyObject
[pylong]: https://docs.python.org/3/c-api/long.html#c.PyLongObject

The C-API docs suggest that [`PyLong_FromLong`][pylong-from-long] is what we're
after. The [source][PyLong_FromLong] shows its doing a bunch of things,
but the imporant bit is the check for `IS_SMALL_INT`, which makes sure that
`ival` is in the range `-5..256`.

[pylong-from-long]: https://docs.python.org/3/c-api/long.html?highlight=pylong_from#c.PyLong_FromLong
[PyLong_FromLong]: https://github.com/python/cpython/blob/6066f450b91f1cbebf33a245c14e660052ccd90a/Objects/longobject.c#L284-L321

Here's an excerpt:

```c
/* In some other header(s) */
#define _PY_NSMALLPOSINTS 257
#define _PY_NSMALLNEGINTS 5
#define IS_SMALL_INT(ival) (-_PY_NSMALLNEGINTS <= (ival) && (ival) < _PY_NSMALLPOSINTS)

/* longobject.c */
PyObject *
PyLong_FromLong(long ival)
{
    /* Handle small and medium cases:
       Numbers from -5 through 256 end up here. Pulled from a pool of
       pre-allocated "longobjects".
     */
    if (IS_SMALL_INT(ival)) {
        return get_small_int((sdigit)ival);
    }
    /* Check medium ints */
    /* If neither small nor medium, allocate a new one and return it */
}
```

Python takes a `long` from the machine and turns that into a `PyLongObject` if
it falls in the (inclusive) range: `[5, 256]`. No matter how many times a value
from this range is requested, the same value gets returned. This optimizes space
for smaller integral constants peppered throughout most programs,
[some code for the curious][integer-pool]. No matter how many times you "create"
`0` the same value is returned. This can be observed using the
[`id(...)`][id-fn] builtin in python.


```python
>>> x = 0
>>> y = 0
>>> id(0)
4400136400
>>> id(x)
4400136400
>>> id(y)
4400136400
>>> x is y
True
```

And now for something different:

```python
>>> id(257)
440297102
>>> x = 257
>>> id(x)
4402972112
>>> y = 257
>>> id(y)
440297217
>>> x is y
False
```
<!--

Quiz: Why does this happen? Tuple assignment seems to be special about fetching
memory

```python
>>> # Apparently tuple assignment is special?
>>> x, y = 257, 257
>>> x is y
True
```
-->

There's still one missing piece: What does the returned `PyLongObject` look like?

[integer-pool]: https://github.com/python/cpython/blob/7ad6f74fcf9db1ccfeaf0986064870d8d3887300/Include/internal/pycore_runtime_init.h#L127-L390
[id-fn]: https://docs.python.org/3/library/functions.html#id

### `PyLongObject` a.k.a. [`struct _longobject`][longintrepr]

A lot of digging (and fighting with GitHub's code search) later, I found that
CPython uses some kind of preamble include so its not easy to jump around the
code, so here are links to my future self:

* Most of the structure is defined here: [object.h][pyobject_var_head]
* `PyLongObject` structure defined here: [longintrepr.h][longintrepr]
* Convenience typedefs: [pytypedefs.h][pylongobject_typedef]


```c
typedef ssize_t Py_ssize_t;

/* From pytypedefs.h */
typedef struct _object PyObject;
typedef struct _longobject PyLongObject;
typedef struct _typeobject PyTypeObject;

/* This chonker:
 * https://github.com/python/cpython/blob/677320348728ce058fa3579017e985af74a236d4/Include/cpython/object.h#L148-L230
 */
struct _typeobject { /* not important to us */ };

/* Defines pointers to support a doubly-linked list of all live heap objects
   when tracing is enabled. (Disabled by default) */
#ifdef Py_TRACE_REFS

#define _PyObject_HEAD_EXTRA \
    PyObject *_ob_next;      \
    PyObject *_ob_prev;
#else
#define _PyObject_HEAD_EXTRA
#endif

struct _object {
    _PyObject_HEAD_EXTRA // This macro is empty by default
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
};

typedef uint32_t digit;

typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;

#define PyObject_VAR_HEAD      PyVarObject ob_base;

struct _longobject {
    PyObject_VAR_HEAD
    digit ob_digit[1];
};

/* From object.h */
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;

#define PyObject_VAR_HEAD  PyVarObject ob_base;

/* From: longintrepr.h: */
struct _longobject {
    PyObject_VAR_HEAD
    digit ob_digit[1];
};

/* From: pytypedefs.h */
typedef struct _longobject PyLongObject
```

Phew! That took a while to dig up using GitHub code search.

### Calculating the Size of `0`

If we were to flatten out the structure by merging all the structs together,
it comes out to be:

```c
struct _longobject {
    Py_ssize_t ob_refcnt;   // From: PyObject_VAR_HEAD -> ob_base
    PyTypeObject *ob_type;  // From: PyObject_VAR_HEAD -> ob_base
    Py_ssize_t ob_size;     // From: PyObject_VAR_HEAD
    digit ob_digit[1];      // Original field: Used as variable size storage
}
```

That totals out to:

| Field       | Type                        | Size on my machine |
| :---        | :---                        | ---: |
| `ob_refcnt` | `ssize_t`                   | 8    |
| `ob_type`   | `void *`[^void-star-note]   | 8    |
| `ob_size`   | `ssize_t`                   | 8    |
| `ob_digit`  | `uint32_t[1]`               | 4    |
| **TOTAL**   | ---                         | 28   |

[^void-star-note]:
    I'm using `void *` because I'm assuming that its safe to cast ob_type from
    `PyTypeObject *` to `void *` and back. This assumption is key in letting us
    calculate the size without caring about the actual type of the raw pointer.


Almost there! This suggests that all numbers below `2**32` occupy `28` bytes.

Python's REPL suggested that `0` is 4 bytes smaller. The magic lies in how
how [`__sizeof__`](int_sizeof) calculates the size.
`Py_SIZE` is shorthand for accessing the `ob_size` field in `PyLongObject`.
`0` is special because it's `ob_size` is `0`.
This is why 0 is reported as taking up 4 fewer bytes than `1`[^sizeof-zero-bug].

Because `PyLongObject` is a variable sized object, a simple C `sizeof(...)`
wouldn't always be correct. The right answer would be to add up the fixed and
variable sized components separately.

```c
int___sizeof___impl(PyObject *self)
{
    Py_ssize_t res;
    /*    SPACE_TAKEN_UP_BY_FIXED_FIELDS   + SPACE_FOR_VARIABLE_SIZE_FIELDS    */
    res = offsetof(PyLongObject, ob_digit) + Py_ABS(Py_SIZE(self))*sizeof(digit);
    return res;
}
```

[^sizeof-zero-bug]: I've reported a bug, waiting for response/discussion [here](https://github.com/python/cpython/issues/98159)

### Things that don't add up

The zero singleton is defined through [`_PyLong_DIGIT_INIT(0)`][zero_singleton].
However, what is confusing is that [`_PyLong_DIGIT_INIT`][pylong_digit_init]
still allocates (stack) space for the `ob_digit` field.

Does this mean that `0` should actually occupy `28` bytes, and not `24` bytes?

Is there something I'm missing?

[pyobject_var_head]: https://github.com/python/cpython/blob/026ab6f4e50658b798be8d1ccf4f3005966e33ea/Include/object.h#L78-L114
[longintrepr]: https://github.com/python/cpython/blob/863729e9c6f599286f98ec37c8716e982c4ca9dd/Include/cpython/longintrepr.h#L79-L82
[pylongobject_typedef]: https://github.com/python/cpython/blob/0b63215bb152c06404cecbd5303b1a50969a9f9f/Include/pytypedefs.h#L18-L20
[int_sizeof]: https://github.com/python/cpython/blob/07b8e85d0e29bc59a7a7d7d662db500c93980edb/Objects/longobject.c#L5732-L5739
[zero_singleton]: https://github.com/python/cpython/blob/4216dce04b7d3f329beaaafc82a77c4ac6cf4d57/Include/internal/pycore_runtime_init.h#L134
[pylong_digit_init]: https://github.com/python/cpython/blob/4216dce04b7d3f329beaaafc82a77c4ac6cf4d57/Include/internal/pycore_runtime_init.h#L78


## Concluding Thoughts

In this long-winded article, we took a look at how Python represents
("small" through "medium" sized) numbers internally. The benefit of metadata like
size, typeinfo, refcounts in `struct _longobject` was what added to a lot of the
"cost" of an integer in Python.

### Playground Link

Here's a link to a compiler explorer playground with the relevant code:
https://godbolt.org/z/rx7PbEYxd

### TL;DR

Python represents integers in base `2**32`, annotating them with information
like refcounts, size, type information and this is what accounts for the extra
size.
