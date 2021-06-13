### Include/listobject.h

```c

/* List object interface */

/*
Another generally useful object type is a list of object pointers.
This is a mutable type: the list items can be changed, and items can be
added or removed.  Out-of-range indices or non-list objects are ignored.

*** WARNING *** PyList_SetItem does not increment the new item's reference
count, but does decrement the reference count of the item it replaces,
if not nil.  It does *decrement* the reference count if it is *not*
inserted in the list.  Similarly, PyList_GetItem does not increment the
returned item's reference count.
*/

#ifndef Py_LISTOBJECT_H
#define Py_LISTOBJECT_H
#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item;

    /* ob_item contains space for 'allocated' elements.  The number
     * currently in use is ob_size.
     * Invariants:
     *     0 <= ob_size <= allocated
     *     len(list) == ob_size
     *     ob_item == NULL implies ob_size == allocated == 0
     * list.sort() temporarily sets allocated to -1 to detect mutations.
     *
     * Items must normally not be NULL, except during construction when
     * the list is not yet visible outside the function that builds it.
     */
    Py_ssize_t allocated;
} PyListObject;

PyAPI_DATA(PyTypeObject) PyList_Type;

#define PyList_Check(op) \
    PyType_FastSubclass(Py_TYPE(op), Py_TPFLAGS_LIST_SUBCLASS)
#define PyList_CheckExact(op) (Py_TYPE(op) == &PyList_Type)

PyAPI_FUNC(PyObject *) PyList_New(Py_ssize_t size);
PyAPI_FUNC(Py_ssize_t) PyList_Size(PyObject *);
PyAPI_FUNC(PyObject *) PyList_GetItem(PyObject *, Py_ssize_t);
PyAPI_FUNC(int) PyList_SetItem(PyObject *, Py_ssize_t, PyObject *);
PyAPI_FUNC(int) PyList_Insert(PyObject *, Py_ssize_t, PyObject *);
PyAPI_FUNC(int) PyList_Append(PyObject *, PyObject *);
PyAPI_FUNC(PyObject *) PyList_GetSlice(PyObject *, Py_ssize_t, Py_ssize_t);
PyAPI_FUNC(int) PyList_SetSlice(PyObject *, Py_ssize_t, Py_ssize_t, PyObject *);
PyAPI_FUNC(int) PyList_Sort(PyObject *);
PyAPI_FUNC(int) PyList_Reverse(PyObject *);
PyAPI_FUNC(PyObject *) PyList_AsTuple(PyObject *);
PyAPI_FUNC(PyObject *) _PyList_Extend(PyListObject *, PyObject *);

/* Macro, trading safety for speed */
#define PyList_GET_ITEM(op, i) (((PyListObject *)(op))->ob_item[i])
#define PyList_SET_ITEM(op, i, v) (((PyListObject *)(op))->ob_item[i] = (v))
#define PyList_GET_SIZE(op)    Py_SIZE(op)

#ifdef __cplusplus
}
#endif
#endif /* !Py_LISTOBJECT_H */
```

### Object/listobject.c

```c
/* List object implementation */

#include "Python.h"

#ifdef STDC_HEADERS
#include <stddef.h>
#else
#include <sys/types.h>          /* For size_t */
#endif

/* Ensure ob_item has room for at least newsize elements, and set
 * ob_size to newsize.  If newsize > ob_size on entry, the content
 * of the new slots at exit is undefined heap trash; it's the caller's
 * responsibility to overwrite them with sane values.
 * The number of allocated elements may grow, shrink, or stay the same.
 * Failure is impossible if newsize <= self.allocated on entry, although
 * that partly relies on an assumption that the system realloc() never
 * fails when passed a number of bytes <= the number of bytes last
 * allocated (the C standard doesn't guarantee this, but it's hard to
 * imagine a realloc implementation where it wouldn't be true).
 * Note that self->ob_item may change, and even if newsize is less
 * than ob_size on entry.
 */
static int
list_resize(PyListObject *self, Py_ssize_t newsize)
{
    PyObject **items;
    size_t new_allocated;
    Py_ssize_t allocated = self->allocated;

    /* Bypass realloc() when a previous overallocation is large enough
       to accommodate the newsize.  If the newsize falls lower than half
       the allocated size, then proceed with the realloc() to shrink the list.
    */
    if (allocated >= newsize && newsize >= (allocated >> 1)) {
        assert(self->ob_item != NULL || newsize == 0);
        Py_SIZE(self) = newsize;
        return 0;
    }

    /* This over-allocates proportional to the list size, making room
     * for additional growth.  The over-allocation is mild, but is
     * enough to give linear-time amortized behavior over a long
     * sequence of appends() in the presence of a poorly-performing
     * system realloc().
     * The growth pattern is:  0, 4, 8, 16, 25, 35, 46, 58, 72, 88, ...
     */
    new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6);

    /* check for integer overflow */
    if (new_allocated > PY_SIZE_MAX - newsize) {
        PyErr_NoMemory();
        return -1;
    } else {
        new_allocated += newsize;
    }

    if (newsize == 0)
        new_allocated = 0;
    items = self->ob_item;
    if (new_allocated <= (PY_SIZE_MAX / sizeof(PyObject *)))
        PyMem_RESIZE(items, PyObject *, new_allocated);
    else
        items = NULL;
    if (items == NULL) {
        PyErr_NoMemory();
        return -1;
    }
    self->ob_item = items;
    Py_SIZE(self) = newsize;
    self->allocated = new_allocated;
    return 0;
}

/* Debug statistic to compare allocations with reuse through the free list */
#undef SHOW_ALLOC_COUNT
#ifdef SHOW_ALLOC_COUNT
static size_t count_alloc = 0;
static size_t count_reuse = 0;

static void
show_alloc(void)
{
    fprintf(stderr, "List allocations: %" PY_FORMAT_SIZE_T "d\n",
        count_alloc);
    fprintf(stderr, "List reuse through freelist: %" PY_FORMAT_SIZE_T
        "d\n", count_reuse);
    fprintf(stderr, "%.2f%% reuse rate\n\n",
        (100.0*count_reuse/(count_alloc+count_reuse)));
}
#endif

/* Empty list reuse scheme to save calls to malloc and free */
#ifndef PyList_MAXFREELIST
#define PyList_MAXFREELIST 80
#endif
static PyListObject *free_list[PyList_MAXFREELIST];
static int numfree = 0;

void
PyList_Fini(void)
{
    PyListObject *op;

    while (numfree) {
        op = free_list[--numfree];
        assert(PyList_CheckExact(op));
        PyObject_GC_Del(op);
    }
}

PyObject *
PyList_New(Py_ssize_t size)
{
    PyListObject *op;
    size_t nbytes;
#ifdef SHOW_ALLOC_COUNT
    static int initialized = 0;
    if (!initialized) {
        Py_AtExit(show_alloc);
        initialized = 1;
    }
#endif

    if (size < 0) {
        PyErr_BadInternalCall();
        return NULL;
    }
    /* Check for overflow without an actual overflow,
     *  which can cause compiler to optimise out */
    if ((size_t)size > PY_SIZE_MAX / sizeof(PyObject *))
        return PyErr_NoMemory();
    nbytes = size * sizeof(PyObject *);
    if (numfree) {
        numfree--;
        op = free_list[numfree];
        _Py_NewReference((PyObject *)op);
#ifdef SHOW_ALLOC_COUNT
        count_reuse++;
#endif
    } else {
        op = PyObject_GC_New(PyListObject, &PyList_Type);
        if (op == NULL)
            return NULL;
#ifdef SHOW_ALLOC_COUNT
        count_alloc++;
#endif
    }
    if (size <= 0)
        op->ob_item = NULL;
    else {
        op->ob_item = (PyObject **) PyMem_MALLOC(nbytes);
        if (op->ob_item == NULL) {
            Py_DECREF(op);
            return PyErr_NoMemory();
        }
        memset(op->ob_item, 0, nbytes);
    }
    Py_SIZE(op) = size;
    op->allocated = size;
    _PyObject_GC_TRACK(op);
    return (PyObject *) op;
}

Py_ssize_t
PyList_Size(PyObject *op)
{
    if (!PyList_Check(op)) {
        PyErr_BadInternalCall();
        return -1;
    }
    else
        return Py_SIZE(op);
}

static PyObject *indexerr = NULL;

PyObject *
PyList_GetItem(PyObject *op, Py_ssize_t i)
{
    if (!PyList_Check(op)) {
        PyErr_BadInternalCall();
        return NULL;
    }
    if (i < 0 || i >= Py_SIZE(op)) {
        if (indexerr == NULL) {
            indexerr = PyString_FromString(
                "list index out of range");
            if (indexerr == NULL)
                return NULL;
        }
        PyErr_SetObject(PyExc_IndexError, indexerr);
        return NULL;
    }
    return ((PyListObject *)op) -> ob_item[i];
}

int
PyList_SetItem(register PyObject *op, register Py_ssize_t i,
               register PyObject *newitem)
{
    register PyObject *olditem;
    register PyObject **p;
    if (!PyList_Check(op)) {
        Py_XDECREF(newitem);
        PyErr_BadInternalCall();
        return -1;
    }
    if (i < 0 || i >= Py_SIZE(op)) {
        Py_XDECREF(newitem);
        PyErr_SetString(PyExc_IndexError,
                        "list assignment index out of range");
        return -1;
    }
    p = ((PyListObject *)op) -> ob_item + i;
    olditem = *p;
    *p = newitem;
    Py_XDECREF(olditem);
    return 0;
}

static int
ins1(PyListObject *self, Py_ssize_t where, PyObject *v)
{
    Py_ssize_t i, n = Py_SIZE(self);
    PyObject **items;
    if (v == NULL) {
        PyErr_BadInternalCall();
        return -1;
    }
    if (n == PY_SSIZE_T_MAX) {
        PyErr_SetString(PyExc_OverflowError,
            "cannot add more objects to list");
        return -1;
    }

    if (list_resize(self, n+1) == -1)
        return -1;

    if (where < 0) {
        where += n;
        if (where < 0)
            where = 0;
    }
    if (where > n)
        where = n;
    items = self->ob_item;
    for (i = n; --i >= where; )
        items[i+1] = items[i];
    Py_INCREF(v);
    items[where] = v;
    return 0;
}

int
PyList_Insert(PyObject *op, Py_ssize_t where, PyObject *newitem)
{
    if (!PyList_Check(op)) {
        PyErr_BadInternalCall();
        return -1;
    }
    return ins1((PyListObject *)op, where, newitem);
}

static int
app1(PyListObject *self, PyObject *v)
{
    Py_ssize_t n = PyList_GET_SIZE(self);

    assert (v != NULL);
    if (n == PY_SSIZE_T_MAX) {
        PyErr_SetString(PyExc_OverflowError,
            "cannot add more objects to list");
        return -1;
    }

    if (list_resize(self, n+1) == -1)
        return -1;

    Py_INCREF(v);
    PyList_SET_ITEM(self, n, v);
    return 0;
}

int
PyList_Append(PyObject *op, PyObject *newitem)
{
    if (PyList_Check(op) && (newitem != NULL))
        return app1((PyListObject *)op, newitem);
    PyErr_BadInternalCall();
    return -1;
}

/* Methods */

static void
list_dealloc(PyListObject *op)
{
    Py_ssize_t i;
    PyObject_GC_UnTrack(op);
    Py_TRASHCAN_SAFE_BEGIN(op)
    if (op->ob_item != NULL) {
        /* Do it backwards, for Christian Tismer.
           There's a simple test case where somehow this reduces
           thrashing when a *very* large list is created and
           immediately deleted. */
        i = Py_SIZE(op);
        while (--i >= 0) {
            Py_XDECREF(op->ob_item[i]);
        }
        PyMem_FREE(op->ob_item);
    }
    if (numfree < PyList_MAXFREELIST && PyList_CheckExact(op))
        free_list[numfree++] = op;
    else
        Py_TYPE(op)->tp_free((PyObject *)op);
    Py_TRASHCAN_SAFE_END(op)
}

static int
list_print(PyListObject *op, FILE *fp, int flags)
{
    int rc;
    Py_ssize_t i;
    PyObject *item;

    rc = Py_ReprEnter((PyObject*)op);
    if (rc != 0) {
        if (rc < 0)
            return rc;
        Py_BEGIN_ALLOW_THREADS
        fprintf(fp, "[...]");
        Py_END_ALLOW_THREADS
        return 0;
    }
    Py_BEGIN_ALLOW_THREADS
    fprintf(fp, "[");
    Py_END_ALLOW_THREADS
    for (i = 0; i < Py_SIZE(op); i++) {
        item = op->ob_item[i];
        Py_INCREF(item);
        if (i > 0) {
            Py_BEGIN_ALLOW_THREADS
            fprintf(fp, ", ");
            Py_END_ALLOW_THREADS
        }
        if (PyObject_Print(item, fp, 0) != 0) {
            Py_DECREF(item);
            Py_ReprLeave((PyObject *)op);
            return -1;
        }
        Py_DECREF(item);
    }
    Py_BEGIN_ALLOW_THREADS
    fprintf(fp, "]");
    Py_END_ALLOW_THREADS
    Py_ReprLeave((PyObject *)op);
    return 0;
}

static PyObject *
list_repr(PyListObject *v)
{
    Py_ssize_t i;
    PyObject *s, *temp;
    PyObject *pieces = NULL, *result = NULL;

    i = Py_ReprEnter((PyObject*)v);
    if (i != 0) {
        return i > 0 ? PyString_FromString("[...]") : NULL;
    }

    if (Py_SIZE(v) == 0) {
        result = PyString_FromString("[]");
        goto Done;
    }

    pieces = PyList_New(0);
    if (pieces == NULL)
        goto Done;

    /* Do repr() on each element.  Note that this may mutate the list,
       so must refetch the list size on each iteration. */
    for (i = 0; i < Py_SIZE(v); ++i) {
        int status;
        s = PyObject_Repr(v->ob_item[i]);
        if (s == NULL)
            goto Done;
        status = PyList_Append(pieces, s);
        Py_DECREF(s);  /* append created a new ref */
        if (status < 0)
            goto Done;
    }

    /* Add "[]" decorations to the first and last items. */
    assert(PyList_GET_SIZE(pieces) > 0);
    s = PyString_FromString("[");
    if (s == NULL)
        goto Done;
    temp = PyList_GET_ITEM(pieces, 0);
    PyString_ConcatAndDel(&s, temp);
    PyList_SET_ITEM(pieces, 0, s);
    if (s == NULL)
        goto Done;

    s = PyString_FromString("]");
    if (s == NULL)
        goto Done;
    temp = PyList_GET_ITEM(pieces, PyList_GET_SIZE(pieces) - 1);
    PyString_ConcatAndDel(&temp, s);
    PyList_SET_ITEM(pieces, PyList_GET_SIZE(pieces) - 1, temp);
    if (temp == NULL)
        goto Done;

    /* Paste them all together with ", " between. */
    s = PyString_FromString(", ");
    if (s == NULL)
        goto Done;
    result = _PyString_Join(s, pieces);
    Py_DECREF(s);

Done:
    Py_XDECREF(pieces);
    Py_ReprLeave((PyObject *)v);
    return result;
}

static Py_ssize_t
list_length(PyListObject *a)
{
    return Py_SIZE(a);
}

static int
list_contains(PyListObject *a, PyObject *el)
{
    Py_ssize_t i;
    int cmp;

    for (i = 0, cmp = 0 ; cmp == 0 && i < Py_SIZE(a); ++i)
        cmp = PyObject_RichCompareBool(el, PyList_GET_ITEM(a, i),
                                           Py_EQ);
    return cmp;
}

static PyObject *
list_item(PyListObject *a, Py_ssize_t i)
{
    if (i < 0 || i >= Py_SIZE(a)) {
        if (indexerr == NULL) {
            indexerr = PyString_FromString(
                "list index out of range");
            if (indexerr == NULL)
                return NULL;
        }
        PyErr_SetObject(PyExc_IndexError, indexerr);
        return NULL;
    }
    Py_INCREF(a->ob_item[i]);
    return a->ob_item[i];
}

static PyObject *
list_slice(PyListObject *a, Py_ssize_t ilow, Py_ssize_t ihigh)
{
    PyListObject *np;
    PyObject **src, **dest;
    Py_ssize_t i, len;
    if (ilow < 0)
        ilow = 0;
    else if (ilow > Py_SIZE(a))
        ilow = Py_SIZE(a);
    if (ihigh < ilow)
        ihigh = ilow;
    else if (ihigh > Py_SIZE(a))
        ihigh = Py_SIZE(a);
    len = ihigh - ilow;
    np = (PyListObject *) PyList_New(len);
    if (np == NULL)
        return NULL;

    src = a->ob_item + ilow;
    dest = np->ob_item;
    for (i = 0; i < len; i++) {
        PyObject *v = src[i];
        Py_INCREF(v);
        dest[i] = v;
    }
    return (PyObject *)np;
}

PyObject *
PyList_GetSlice(PyObject *a, Py_ssize_t ilow, Py_ssize_t ihigh)
{
    if (!PyList_Check(a)) {
        PyErr_BadInternalCall();
        return NULL;
    }
    return list_slice((PyListObject *)a, ilow, ihigh);
}

static PyObject *
list_concat(PyListObject *a, PyObject *bb)
{
    Py_ssize_t size;
    Py_ssize_t i;
    PyObject **src, **dest;
    PyListObject *np;
    if (!PyList_Check(bb)) {
        PyErr_Format(PyExc_TypeError,
                  "can only concatenate list (not \"%.200s\") to list",
                  bb->ob_type->tp_name);
        return NULL;
    }
#define b ((PyListObject *)bb)
    size = Py_SIZE(a) + Py_SIZE(b);
    if (size < 0)
        return PyErr_NoMemory();
    np = (PyListObject *) PyList_New(size);
    if (np == NULL) {
        return NULL;
    }
    src = a->ob_item;
    dest = np->ob_item;
    for (i = 0; i < Py_SIZE(a); i++) {
        PyObject *v = src[i];
        Py_INCREF(v);
        dest[i] = v;
    }
    src = b->ob_item;
    dest = np->ob_item + Py_SIZE(a);
    for (i = 0; i < Py_SIZE(b); i++) {
        PyObject *v = src[i];
        Py_INCREF(v);
        dest[i] = v;
    }
    return (PyObject *)np;
#undef b
}

static PyObject *
list_repeat(PyListObject *a, Py_ssize_t n)
{
    Py_ssize_t i, j;
    Py_ssize_t size;
    PyListObject *np;
    PyObject **p, **items;
    PyObject *elem;
    if (n < 0)
        n = 0;
    if (n > 0 && Py_SIZE(a) > PY_SSIZE_T_MAX / n)
        return PyErr_NoMemory();
    size = Py_SIZE(a) * n;
    if (size == 0)
        return PyList_New(0);
    np = (PyListObject *) PyList_New(size);
    if (np == NULL)
        return NULL;

    items = np->ob_item;
    if (Py_SIZE(a) == 1) {
        elem = a->ob_item[0];
        for (i = 0; i < n; i++) {
            items[i] = elem;
            Py_INCREF(elem);
        }
        return (PyObject *) np;
    }
    p = np->ob_item;
    items = a->ob_item;
    for (i = 0; i < n; i++) {
        for (j = 0; j < Py_SIZE(a); j++) {
            *p = items[j];
            Py_INCREF(*p);
            p++;
        }
    }
    return (PyObject *) np;
}

static int
list_clear(PyListObject *a)
{
    Py_ssize_t i;
    PyObject **item = a->ob_item;
    if (item != NULL) {
        /* Because XDECREF can recursively invoke operations on
           this list, we make it empty first. */
        i = Py_SIZE(a);
        Py_SIZE(a) = 0;
        a->ob_item = NULL;
        a->allocated = 0;
        while (--i >= 0) {
            Py_XDECREF(item[i]);
        }
        PyMem_FREE(item);
    }
    /* Never fails; the return value can be ignored.
       Note that there is no guarantee that the list is actually empty
       at this point, because XDECREF may have populated it again! */
    return 0;
}

/* a[ilow:ihigh] = v if v != NULL.
 * del a[ilow:ihigh] if v == NULL.
 *
 * Special speed gimmick:  when v is NULL and ihigh - ilow <= 8, it's
 * guaranteed the call cannot fail.
 */
static int
list_ass_slice(PyListObject *a, Py_ssize_t ilow, Py_ssize_t ihigh, PyObject *v)
{
    /* Because [X]DECREF can recursively invoke list operations on
       this list, we must postpone all [X]DECREF activity until
       after the list is back in its canonical shape.  Therefore
       we must allocate an additional array, 'recycle', into which
       we temporarily copy the items that are deleted from the
       list. :-( */
    PyObject *recycle_on_stack[8];
    PyObject **recycle = recycle_on_stack; /* will allocate more if needed */
    PyObject **item;
    PyObject **vitem = NULL;
    PyObject *v_as_SF = NULL; /* PySequence_Fast(v) */
    Py_ssize_t n; /* # of elements in replacement list */
    Py_ssize_t norig; /* # of elements in list getting replaced */
    Py_ssize_t d; /* Change in size */
    Py_ssize_t k;
    size_t s;
    int result = -1;            /* guilty until proved innocent */
#define b ((PyListObject *)v)
    if (v == NULL)
        n = 0;
    else {
        if (a == b) {
            /* Special case "a[i:j] = a" -- copy b first */
            v = list_slice(b, 0, Py_SIZE(b));
            if (v == NULL)
                return result;
            result = list_ass_slice(a, ilow, ihigh, v);
            Py_DECREF(v);
            return result;
        }
        v_as_SF = PySequence_Fast(v, "can only assign an iterable");
        if(v_as_SF == NULL)
            goto Error;
        n = PySequence_Fast_GET_SIZE(v_as_SF);
        vitem = PySequence_Fast_ITEMS(v_as_SF);
    }
    if (ilow < 0)
        ilow = 0;
    else if (ilow > Py_SIZE(a))
        ilow = Py_SIZE(a);

    if (ihigh < ilow)
        ihigh = ilow;
    else if (ihigh > Py_SIZE(a))
        ihigh = Py_SIZE(a);

    norig = ihigh - ilow;
    assert(norig >= 0);
    d = n - norig;
    if (Py_SIZE(a) + d == 0) {
        Py_XDECREF(v_as_SF);
        return list_clear(a);
    }
    item = a->ob_item;
    /* recycle the items that we are about to remove */
    s = norig * sizeof(PyObject *);
    /* If norig == 0, item might be NULL, in which case we may not memcpy from it. */
    if (s) {
        if (s > sizeof(recycle_on_stack)) {
            recycle = (PyObject **)PyMem_MALLOC(s);
            if (recycle == NULL) {
                PyErr_NoMemory();
                goto Error;
            }
        }
        memcpy(recycle, &item[ilow], s);
    }

    if (d < 0) { /* Delete -d items */
        memmove(&item[ihigh+d], &item[ihigh],
            (Py_SIZE(a) - ihigh)*sizeof(PyObject *));
        list_resize(a, Py_SIZE(a) + d);
        item = a->ob_item;
    }
    else if (d > 0) { /* Insert d items */
        k = Py_SIZE(a);
        if (list_resize(a, k+d) < 0)
            goto Error;
        item = a->ob_item;
        memmove(&item[ihigh+d], &item[ihigh],
            (k - ihigh)*sizeof(PyObject *));
    }
    for (k = 0; k < n; k++, ilow++) {
        PyObject *w = vitem[k];
        Py_XINCREF(w);
        item[ilow] = w;
    }
    for (k = norig - 1; k >= 0; --k)
        Py_XDECREF(recycle[k]);
    result = 0;
 Error:
    if (recycle != recycle_on_stack)
        PyMem_FREE(recycle);
    Py_XDECREF(v_as_SF);
    return result;
#undef b
}

int
PyList_SetSlice(PyObject *a, Py_ssize_t ilow, Py_ssize_t ihigh, PyObject *v)
{
    if (!PyList_Check(a)) {
        PyErr_BadInternalCall();
        return -1;
    }
    return list_ass_slice((PyListObject *)a, ilow, ihigh, v);
}

static PyObject *
list_inplace_repeat(PyListObject *self, Py_ssize_t n)
{
    PyObject **items;
    Py_ssize_t size, i, j, p;


    size = PyList_GET_SIZE(self);
    if (size == 0 || n == 1) {
        Py_INCREF(self);
        return (PyObject *)self;
    }

    if (n < 1) {
        (void)list_clear(self);
        Py_INCREF(self);
        return (PyObject *)self;
    }

    if (size > PY_SSIZE_T_MAX / n) {
        return PyErr_NoMemory();
    }

    if (list_resize(self, size*n) == -1)
        return NULL;

    p = size;
    items = self->ob_item;
    for (i = 1; i < n; i++) { /* Start counting at 1, not 0 */
        for (j = 0; j < size; j++) {
            PyObject *o = items[j];
            Py_INCREF(o);
            items[p++] = o;
        }
    }
    Py_INCREF(self);
    return (PyObject *)self;
}

static int
list_ass_item(PyListObject *a, Py_ssize_t i, PyObject *v)
{
    PyObject *old_value;
    if (i < 0 || i >= Py_SIZE(a)) {
        PyErr_SetString(PyExc_IndexError,
                        "list assignment index out of range");
        return -1;
    }
    if (v == NULL)
        return list_ass_slice(a, i, i+1, v);
    Py_INCREF(v);
    old_value = a->ob_item[i];
    a->ob_item[i] = v;
    Py_DECREF(old_value);
    return 0;
}

static PyObject *
listinsert(PyListObject *self, PyObject *args)
{
    Py_ssize_t i;
    PyObject *v;
    if (!PyArg_ParseTuple(args, "nO:insert", &i, &v))
        return NULL;
    if (ins1(self, i, v) == 0)
        Py_RETURN_NONE;
    return NULL;
}

static PyObject *
listappend(PyListObject *self, PyObject *v)
{
    if (app1(self, v) == 0)
        Py_RETURN_NONE;
    return NULL;
}

static PyObject *
listextend(PyListObject *self, PyObject *b)
{
    PyObject *it;      /* iter(v) */
    Py_ssize_t m;                  /* size of self */
    Py_ssize_t n;                  /* guess for size of b */
    Py_ssize_t mn;                 /* m + n */
    Py_ssize_t i;
    PyObject *(*iternext)(PyObject *);

    /* Special cases:
       1) lists and tuples which can use PySequence_Fast ops
       2) extending self to self requires making a copy first
    */
    if (PyList_CheckExact(b) || PyTuple_CheckExact(b) || (PyObject *)self == b) {
        PyObject **src, **dest;
        b = PySequence_Fast(b, "argument must be iterable");
        if (!b)
            return NULL;
        n = PySequence_Fast_GET_SIZE(b);
        if (n == 0) {
            /* short circuit when b is empty */
            Py_DECREF(b);
            Py_RETURN_NONE;
        }
        m = Py_SIZE(self);
        if (list_resize(self, m + n) == -1) {
            Py_DECREF(b);
            return NULL;
        }
        /* note that we may still have self == b here for the
         * situation a.extend(a), but the following code works
         * in that case too.  Just make sure to resize self
         * before calling PySequence_Fast_ITEMS.
         */
        /* populate the end of self with b's items */
        src = PySequence_Fast_ITEMS(b);
        dest = self->ob_item + m;
        for (i = 0; i < n; i++) {
            PyObject *o = src[i];
            Py_INCREF(o);
            dest[i] = o;
        }
        Py_DECREF(b);
        Py_RETURN_NONE;
    }

    it = PyObject_GetIter(b);
    if (it == NULL)
        return NULL;
    iternext = *it->ob_type->tp_iternext;

    /* Guess a result list size. */
    n = _PyObject_LengthHint(b, 8);
    if (n == -1) {
        Py_DECREF(it);
        return NULL;
    }
    m = Py_SIZE(self);
    mn = m + n;
    if (mn >= m) {
        /* Make room. */
        if (list_resize(self, mn) == -1)
            goto error;
        /* Make the list sane again. */
        Py_SIZE(self) = m;
    }
    /* Else m + n overflowed; on the chance that n lied, and there really
     * is enough room, ignore it.  If n was telling the truth, we'll
     * eventually run out of memory during the loop.
     */

    /* Run iterator to exhaustion. */
    for (;;) {
        PyObject *item = iternext(it);
        if (item == NULL) {
            if (PyErr_Occurred()) {
                if (PyErr_ExceptionMatches(PyExc_StopIteration))
                    PyErr_Clear();
                else
                    goto error;
            }
            break;
        }
        if (Py_SIZE(self) < self->allocated) {
            /* steals ref */
            PyList_SET_ITEM(self, Py_SIZE(self), item);
            ++Py_SIZE(self);
        }
        else {
            int status = app1(self, item);
            Py_DECREF(item);  /* append creates a new ref */
            if (status < 0)
                goto error;
        }
    }

    /* Cut back result list if initial guess was too large. */
    if (Py_SIZE(self) < self->allocated)
        list_resize(self, Py_SIZE(self));  /* shrinking can't fail */

    Py_DECREF(it);
    Py_RETURN_NONE;

  error:
    Py_DECREF(it);
    return NULL;
}

PyObject *
_PyList_Extend(PyListObject *self, PyObject *b)
{
    return listextend(self, b);
}

static PyObject *
list_inplace_concat(PyListObject *self, PyObject *other)
{
    PyObject *result;

    result = listextend(self, other);
    if (result == NULL)
        return result;
    Py_DECREF(result);
    Py_INCREF(self);
    return (PyObject *)self;
}

static PyObject *
listpop(PyListObject *self, PyObject *args)
{
    Py_ssize_t i = -1;
    PyObject *v;
    int status;

    if (!PyArg_ParseTuple(args, "|n:pop", &i))
        return NULL;

    if (Py_SIZE(self) == 0) {
        /* Special-case most common failure cause */
        PyErr_SetString(PyExc_IndexError, "pop from empty list");
        return NULL;
    }
    if (i < 0)
        i += Py_SIZE(self);
    if (i < 0 || i >= Py_SIZE(self)) {
        PyErr_SetString(PyExc_IndexError, "pop index out of range");
        return NULL;
    }
    v = self->ob_item[i];
    if (i == Py_SIZE(self) - 1) {
        status = list_resize(self, Py_SIZE(self) - 1);
        assert(status >= 0);
        return v; /* and v now owns the reference the list had */
    }
    Py_INCREF(v);
    status = list_ass_slice(self, i, i+1, (PyObject *)NULL);
    assert(status >= 0);
    /* Use status, so that in a release build compilers don't
     * complain about the unused name.
     */
    (void) status;

    return v;
}

/* Reverse a slice of a list in place, from lo up to (exclusive) hi. */
static void
reverse_slice(PyObject **lo, PyObject **hi)
{
    assert(lo && hi);

    --hi;
    while (lo < hi) {
        PyObject *t = *lo;
        *lo = *hi;
        *hi = t;
        ++lo;
        --hi;
    }
}

/* Lots of code for an adaptive, stable, natural mergesort.  There are many
 * pieces to this algorithm; read listsort.txt for overviews and details.
 */

/* Comparison function.  Takes care of calling a user-supplied
 * comparison function (any callable Python object), which must not be
 * NULL (use the ISLT macro if you don't know, or call PyObject_RichCompareBool
 * with Py_LT if you know it's NULL).
 * Returns -1 on error, 1 if x < y, 0 if x >= y.
 */
static int
islt(PyObject *x, PyObject *y, PyObject *compare)
{
    PyObject *res;
    PyObject *args;
    Py_ssize_t i;

    assert(compare != NULL);
    /* Call the user's comparison function and translate the 3-way
     * result into true or false (or error).
     */
    args = PyTuple_New(2);
    if (args == NULL)
        return -1;
    Py_INCREF(x);
    Py_INCREF(y);
    PyTuple_SET_ITEM(args, 0, x);
    PyTuple_SET_ITEM(args, 1, y);
    res = PyObject_Call(compare, args, NULL);
    Py_DECREF(args);
    if (res == NULL)
        return -1;
    if (!PyInt_Check(res)) {
        PyErr_Format(PyExc_TypeError,
                     "comparison function must return int, not %.200s",
                     res->ob_type->tp_name);
        Py_DECREF(res);
        return -1;
    }
    i = PyInt_AsLong(res);
    Py_DECREF(res);
    return i < 0;
}

/* If COMPARE is NULL, calls PyObject_RichCompareBool with Py_LT, else calls
 * islt.  This avoids a layer of function call in the usual case, and
 * sorting does many comparisons.
 * Returns -1 on error, 1 if x < y, 0 if x >= y.
 */
#define ISLT(X, Y, COMPARE) ((COMPARE) == NULL ?                        \
                 PyObject_RichCompareBool(X, Y, Py_LT) :                \
                 islt(X, Y, COMPARE))

/* Compare X to Y via "<".  Goto "fail" if the comparison raises an
   error.  Else "k" is set to true iff X<Y, and an "if (k)" block is
   started.  It makes more sense in context <wink>.  X and Y are PyObject*s.
*/
#define IFLT(X, Y) if ((k = ISLT(X, Y, compare)) < 0) goto fail;  \
           if (k)

/* binarysort is the best method for sorting small arrays: it does
   few compares, but can do data movement quadratic in the number of
   elements.
   [lo, hi) is a contiguous slice of a list, and is sorted via
   binary insertion.  This sort is stable.
   On entry, must have lo <= start <= hi, and that [lo, start) is already
   sorted (pass start == lo if you don't know!).
   If islt() complains return -1, else 0.
   Even in case of error, the output slice will be some permutation of
   the input (nothing is lost or duplicated).
*/
static int
binarysort(PyObject **lo, PyObject **hi, PyObject **start, PyObject *compare)
     /* compare -- comparison function object, or NULL for default */
{
    register Py_ssize_t k;
    register PyObject **l, **p, **r;
    register PyObject *pivot;

    assert(lo <= start && start <= hi);
    /* assert [lo, start) is sorted */
    if (lo == start)
        ++start;
    for (; start < hi; ++start) {
        /* set l to where *start belongs */
        l = lo;
        r = start;
        pivot = *r;
        /* Invariants:
         * pivot >= all in [lo, l).
         * pivot  < all in [r, start).
         * The second is vacuously true at the start.
         */
        assert(l < r);
        do {
            p = l + ((r - l) >> 1);
            IFLT(pivot, *p)
                r = p;
            else
                l = p+1;
        } while (l < r);
        assert(l == r);
        /* The invariants still hold, so pivot >= all in [lo, l) and
           pivot < all in [l, start), so pivot belongs at l.  Note
           that if there are elements equal to pivot, l points to the
           first slot after them -- that's why this sort is stable.
           Slide over to make room.
           Caution: using memmove is much slower under MSVC 5;
           we're not usually moving many slots. */
        for (p = start; p > l; --p)
            *p = *(p-1);
        *l = pivot;
    }
    return 0;

 fail:
    return -1;
}

/*
Return the length of the run beginning at lo, in the slice [lo, hi).  lo < hi
is required on entry.  "A run" is the longest ascending sequence, with

    lo[0] <= lo[1] <= lo[2] <= ...

or the longest descending sequence, with

    lo[0] > lo[1] > lo[2] > ...

Boolean *descending is set to 0 in the former case, or to 1 in the latter.
For its intended use in a stable mergesort, the strictness of the defn of
"descending" is needed so that the caller can safely reverse a descending
sequence without violating stability (strict > ensures there are no equal
elements to get out of order).

Returns -1 in case of error.
*/
static Py_ssize_t
count_run(PyObject **lo, PyObject **hi, PyObject *compare, int *descending)
{
    Py_ssize_t k;
    Py_ssize_t n;

    assert(lo < hi);
    *descending = 0;
    ++lo;
    if (lo == hi)
        return 1;

    n = 2;
    IFLT(*lo, *(lo-1)) {
        *descending = 1;
        for (lo = lo+1; lo < hi; ++lo, ++n) {
            IFLT(*lo, *(lo-1))
                ;
            else
                break;
        }
    }
    else {
        for (lo = lo+1; lo < hi; ++lo, ++n) {
            IFLT(*lo, *(lo-1))
                break;
        }
    }

    return n;
fail:
    return -1;
}

/*
Locate the proper position of key in a sorted vector; if the vector contains
an element equal to key, return the position immediately to the left of
the leftmost equal element.  [gallop_right() does the same except returns
the position to the right of the rightmost equal element (if any).]

"a" is a sorted vector with n elements, starting at a[0].  n must be > 0.

"hint" is an index at which to begin the search, 0 <= hint < n.  The closer
hint is to the final result, the faster this runs.

The return value is the int k in 0..n such that

    a[k-1] < key <= a[k]

pretending that *(a-1) is minus infinity and a[n] is plus infinity.  IOW,
key belongs at index k; or, IOW, the first k elements of a should precede
key, and the last n-k should follow key.

Returns -1 on error.  See listsort.txt for info on the method.
*/
static Py_ssize_t
gallop_left(PyObject *key, PyObject **a, Py_ssize_t n, Py_ssize_t hint, PyObject *compare)
{
    Py_ssize_t ofs;
    Py_ssize_t lastofs;
    Py_ssize_t k;

    assert(key && a && n > 0 && hint >= 0 && hint < n);

    a += hint;
    lastofs = 0;
    ofs = 1;
    IFLT(*a, key) {
        /* a[hint] < key -- gallop right, until
         * a[hint + lastofs] < key <= a[hint + ofs]
         */
        const Py_ssize_t maxofs = n - hint;             /* &a[n-1] is highest */
        while (ofs < maxofs) {
            IFLT(a[ofs], key) {
                lastofs = ofs;
                ofs = (ofs << 1) + 1;
                if (ofs <= 0)                   /* int overflow */
                    ofs = maxofs;
            }
            else                /* key <= a[hint + ofs] */
                break;
        }
        if (ofs > maxofs)
            ofs = maxofs;
        /* Translate back to offsets relative to &a[0]. */
        lastofs += hint;
        ofs += hint;
    }
    else {
        /* key <= a[hint] -- gallop left, until
         * a[hint - ofs] < key <= a[hint - lastofs]
         */
        const Py_ssize_t maxofs = hint + 1;             /* &a[0] is lowest */
        while (ofs < maxofs) {
            IFLT(*(a-ofs), key)
                break;
            /* key <= a[hint - ofs] */
            lastofs = ofs;
            ofs = (ofs << 1) + 1;
            if (ofs <= 0)               /* int overflow */
                ofs = maxofs;
        }
        if (ofs > maxofs)
            ofs = maxofs;
        /* Translate back to positive offsets relative to &a[0]. */
        k = lastofs;
        lastofs = hint - ofs;
        ofs = hint - k;
    }
    a -= hint;

    assert(-1 <= lastofs && lastofs < ofs && ofs <= n);
    /* Now a[lastofs] < key <= a[ofs], so key belongs somewhere to the
     * right of lastofs but no farther right than ofs.  Do a binary
     * search, with invariant a[lastofs-1] < key <= a[ofs].
     */
    ++lastofs;
    while (lastofs < ofs) {
        Py_ssize_t m = lastofs + ((ofs - lastofs) >> 1);

        IFLT(a[m], key)
            lastofs = m+1;              /* a[m] < key */
        else
            ofs = m;                    /* key <= a[m] */
    }
    assert(lastofs == ofs);             /* so a[ofs-1] < key <= a[ofs] */
    return ofs;

fail:
    return -1;
}

/*
Exactly like gallop_left(), except that if key already exists in a[0:n],
finds the position immediately to the right of the rightmost equal value.

The return value is the int k in 0..n such that

    a[k-1] <= key < a[k]

or -1 if error.

The code duplication is massive, but this is enough different given that
we're sticking to "<" comparisons that it's much harder to follow if
written as one routine with yet another "left or right?" flag.
*/
static Py_ssize_t
gallop_right(PyObject *key, PyObject **a, Py_ssize_t n, Py_ssize_t hint, PyObject *compare)
{
    Py_ssize_t ofs;
    Py_ssize_t lastofs;
    Py_ssize_t k;

    assert(key && a && n > 0 && hint >= 0 && hint < n);

    a += hint;
    lastofs = 0;
    ofs = 1;
    IFLT(key, *a) {
        /* key < a[hint] -- gallop left, until
         * a[hint - ofs] <= key < a[hint - lastofs]
         */
        const Py_ssize_t maxofs = hint + 1;             /* &a[0] is lowest */
        while (ofs < maxofs) {
            IFLT(key, *(a-ofs)) {
                lastofs = ofs;
                ofs = (ofs << 1) + 1;
                if (ofs <= 0)                   /* int overflow */
                    ofs = maxofs;
            }
            else                /* a[hint - ofs] <= key */
                break;
        }
        if (ofs > maxofs)
            ofs = maxofs;
        /* Translate back to positive offsets relative to &a[0]. */
        k = lastofs;
        lastofs = hint - ofs;
        ofs = hint - k;
    }
    else {
        /* a[hint] <= key -- gallop right, until
         * a[hint + lastofs] <= key < a[hint + ofs]
        */
        const Py_ssize_t maxofs = n - hint;             /* &a[n-1] is highest */
        while (ofs < maxofs) {
            IFLT(key, a[ofs])
                break;
            /* a[hint + ofs] <= key */
            lastofs = ofs;
            ofs = (ofs << 1) + 1;
            if (ofs <= 0)               /* int overflow */
                ofs = maxofs;
        }
        if (ofs > maxofs)
            ofs = maxofs;
        /* Translate back to offsets relative to &a[0]. */
        lastofs += hint;
        ofs += hint;
    }
    a -= hint;

    assert(-1 <= lastofs && lastofs < ofs && ofs <= n);
    /* Now a[lastofs] <= key < a[ofs], so key belongs somewhere to the
     * right of lastofs but no farther right than ofs.  Do a binary
     * search, with invariant a[lastofs-1] <= key < a[ofs].
     */
    ++lastofs;
    while (lastofs < ofs) {
        Py_ssize_t m = lastofs + ((ofs - lastofs) >> 1);

        IFLT(key, a[m])
            ofs = m;                    /* key < a[m] */
        else
            lastofs = m+1;              /* a[m] <= key */
    }
    assert(lastofs == ofs);             /* so a[ofs-1] <= key < a[ofs] */
    return ofs;

fail:
    return -1;
}

/* The maximum number of entries in a MergeState's pending-runs stack.
 * This is enough to sort arrays of size up to about
 *     32 * phi ** MAX_MERGE_PENDING
 * where phi ~= 1.618.  85 is ridiculouslylarge enough, good for an array
 * with 2**64 elements.
 */
#define MAX_MERGE_PENDING 85

/* When we get into galloping mode, we stay there until both runs win less
 * often than MIN_GALLOP consecutive times.  See listsort.txt for more info.
 */
#define MIN_GALLOP 7

/* Avoid malloc for small temp arrays. */
#define MERGESTATE_TEMP_SIZE 256

/* One MergeState exists on the stack per invocation of mergesort.  It's just
 * a convenient way to pass state around among the helper functions.
 */
struct s_slice {
    PyObject **base;
    Py_ssize_t len;
};

typedef struct s_MergeState {
    /* The user-supplied comparison function. or NULL if none given. */
    PyObject *compare;

    /* This controls when we get *into* galloping mode.  It's initialized
     * to MIN_GALLOP.  merge_lo and merge_hi tend to nudge it higher for
     * random data, and lower for highly structured data.
     */
    Py_ssize_t min_gallop;

    /* 'a' is temp storage to help with merges.  It contains room for
     * alloced entries.
     */
    PyObject **a;       /* may point to temparray below */
    Py_ssize_t alloced;

    /* A stack of n pending runs yet to be merged.  Run #i starts at
     * address base[i] and extends for len[i] elements.  It's always
     * true (so long as the indices are in bounds) that
     *
     *     pending[i].base + pending[i].len == pending[i+1].base
     *
     * so we could cut the storage for this, but it's a minor amount,
     * and keeping all the info explicit simplifies the code.
     */
    int n;
    struct s_slice pending[MAX_MERGE_PENDING];

    /* 'a' points to this when possible, rather than muck with malloc. */
    PyObject *temparray[MERGESTATE_TEMP_SIZE];
} MergeState;

/* Conceptually a MergeState's constructor. */
static void
merge_init(MergeState *ms, PyObject *compare)
{
    assert(ms != NULL);
    ms->compare = compare;
    ms->a = ms->temparray;
    ms->alloced = MERGESTATE_TEMP_SIZE;
    ms->n = 0;
    ms->min_gallop = MIN_GALLOP;
}

/* Free all the temp memory owned by the MergeState.  This must be called
 * when you're done with a MergeState, and may be called before then if
 * you want to free the temp memory early.
 */
static void
merge_freemem(MergeState *ms)
{
    assert(ms != NULL);
    if (ms->a != ms->temparray)
        PyMem_Free(ms->a);
    ms->a = ms->temparray;
    ms->alloced = MERGESTATE_TEMP_SIZE;
}

/* Ensure enough temp memory for 'need' array slots is available.
 * Returns 0 on success and -1 if the memory can't be gotten.
 */
static int
merge_getmem(MergeState *ms, Py_ssize_t need)
{
    assert(ms != NULL);
    if (need <= ms->alloced)
        return 0;
    /* Don't realloc!  That can cost cycles to copy the old data, but
     * we don't care what's in the block.
     */
    merge_freemem(ms);
    if ((size_t)need > PY_SSIZE_T_MAX / sizeof(PyObject*)) {
        PyErr_NoMemory();
        return -1;
    }
    ms->a = (PyObject **)PyMem_Malloc(need * sizeof(PyObject*));
    if (ms->a) {
        ms->alloced = need;
        return 0;
    }
    PyErr_NoMemory();
    merge_freemem(ms);          /* reset to sane state */
    return -1;
}
#define MERGE_GETMEM(MS, NEED) ((NEED) <= (MS)->alloced ? 0 :   \
                                merge_getmem(MS, NEED))

/* Merge the na elements starting at pa with the nb elements starting at pb
 * in a stable way, in-place.  na and nb must be > 0, and pa + na == pb.
 * Must also have that *pb < *pa, that pa[na-1] belongs at the end of the
 * merge, and should have na <= nb.  See listsort.txt for more info.
 * Return 0 if successful, -1 if error.
 */
static Py_ssize_t
merge_lo(MergeState *ms, PyObject **pa, Py_ssize_t na,
                         PyObject **pb, Py_ssize_t nb)
{
    Py_ssize_t k;
    PyObject *compare;
    PyObject **dest;
    int result = -1;            /* guilty until proved innocent */
    Py_ssize_t min_gallop;

    assert(ms && pa && pb && na > 0 && nb > 0 && pa + na == pb);
    if (MERGE_GETMEM(ms, na) < 0)
        return -1;
    memcpy(ms->a, pa, na * sizeof(PyObject*));
    dest = pa;
    pa = ms->a;

    *dest++ = *pb++;
    --nb;
    if (nb == 0)
        goto Succeed;
    if (na == 1)
        goto CopyB;

    min_gallop = ms->min_gallop;
    compare = ms->compare;
    for (;;) {
        Py_ssize_t acount = 0;          /* # of times A won in a row */
        Py_ssize_t bcount = 0;          /* # of times B won in a row */

        /* Do the straightforward thing until (if ever) one run
         * appears to win consistently.
         */
        for (;;) {
            assert(na > 1 && nb > 0);
            k = ISLT(*pb, *pa, compare);
            if (k) {
                if (k < 0)
                    goto Fail;
                *dest++ = *pb++;
                ++bcount;
                acount = 0;
                --nb;
                if (nb == 0)
                    goto Succeed;
                if (bcount >= min_gallop)
                    break;
            }
            else {
                *dest++ = *pa++;
                ++acount;
                bcount = 0;
                --na;
                if (na == 1)
                    goto CopyB;
                if (acount >= min_gallop)
                    break;
            }
        }

        /* One run is winning so consistently that galloping may
         * be a huge win.  So try that, and continue galloping until
         * (if ever) neither run appears to be winning consistently
         * anymore.
         */
        ++min_gallop;
        do {
            assert(na > 1 && nb > 0);
            min_gallop -= min_gallop > 1;
            ms->min_gallop = min_gallop;
            k = gallop_right(*pb, pa, na, 0, compare);
            acount = k;
            if (k) {
                if (k < 0)
                    goto Fail;
                memcpy(dest, pa, k * sizeof(PyObject *));
                dest += k;
                pa += k;
                na -= k;
                if (na == 1)
                    goto CopyB;
                /* na==0 is impossible now if the comparison
                 * function is consistent, but we can't assume
                 * that it is.
                 */
                if (na == 0)
                    goto Succeed;
            }
            *dest++ = *pb++;
            --nb;
            if (nb == 0)
                goto Succeed;

            k = gallop_left(*pa, pb, nb, 0, compare);
            bcount = k;
            if (k) {
                if (k < 0)
                    goto Fail;
                memmove(dest, pb, k * sizeof(PyObject *));
                dest += k;
                pb += k;
                nb -= k;
                if (nb == 0)
                    goto Succeed;
            }
            *dest++ = *pa++;
            --na;
            if (na == 1)
                goto CopyB;
        } while (acount >= MIN_GALLOP || bcount >= MIN_GALLOP);
        ++min_gallop;           /* penalize it for leaving galloping mode */
        ms->min_gallop = min_gallop;
    }
Succeed:
    result = 0;
Fail:
    if (na)
        memcpy(dest, pa, na * sizeof(PyObject*));
    return result;
CopyB:
    assert(na == 1 && nb > 0);
    /* The last element of pa belongs at the end of the merge. */
    memmove(dest, pb, nb * sizeof(PyObject *));
    dest[nb] = *pa;
    return 0;
}

/* Merge the na elements starting at pa with the nb elements starting at pb
 * in a stable way, in-place.  na and nb must be > 0, and pa + na == pb.
 * Must also have that *pb < *pa, that pa[na-1] belongs at the end of the
 * merge, and should have na >= nb.  See listsort.txt for more info.
 * Return 0 if successful, -1 if error.
 */
static Py_ssize_t
merge_hi(MergeState *ms, PyObject **pa, Py_ssize_t na, PyObject **pb, Py_ssize_t nb)
{
    Py_ssize_t k;
    PyObject *compare;
    PyObject **dest;
    int result = -1;            /* guilty until proved innocent */
    PyObject **basea;
    PyObject **baseb;
    Py_ssize_t min_gallop;

    assert(ms && pa && pb && na > 0 && nb > 0 && pa + na == pb);
    if (MERGE_GETMEM(ms, nb) < 0)
        return -1;
    dest = pb + nb - 1;
    memcpy(ms->a, pb, nb * sizeof(PyObject*));
    basea = pa;
    baseb = ms->a;
    pb = ms->a + nb - 1;
    pa += na - 1;

    *dest-- = *pa--;
    --na;
    if (na == 0)
        goto Succeed;
    if (nb == 1)
        goto CopyA;

    min_gallop = ms->min_gallop;
    compare = ms->compare;
    for (;;) {
        Py_ssize_t acount = 0;          /* # of times A won in a row */
        Py_ssize_t bcount = 0;          /* # of times B won in a row */

        /* Do the straightforward thing until (if ever) one run
         * appears to win consistently.
         */
        for (;;) {
            assert(na > 0 && nb > 1);
            k = ISLT(*pb, *pa, compare);
            if (k) {
                if (k < 0)
                    goto Fail;
                *dest-- = *pa--;
                ++acount;
                bcount = 0;
                --na;
                if (na == 0)
                    goto Succeed;
                if (acount >= min_gallop)
                    break;
            }
            else {
                *dest-- = *pb--;
                ++bcount;
                acount = 0;
                --nb;
                if (nb == 1)
                    goto CopyA;
                if (bcount >= min_gallop)
                    break;
            }
        }

        /* One run is winning so consistently that galloping may
         * be a huge win.  So try that, and continue galloping until
         * (if ever) neither run appears to be winning consistently
         * anymore.
         */
        ++min_gallop;
        do {
            assert(na > 0 && nb > 1);
            min_gallop -= min_gallop > 1;
            ms->min_gallop = min_gallop;
            k = gallop_right(*pb, basea, na, na-1, compare);
            if (k < 0)
                goto Fail;
            k = na - k;
            acount = k;
            if (k) {
                dest -= k;
                pa -= k;
                memmove(dest+1, pa+1, k * sizeof(PyObject *));
                na -= k;
                if (na == 0)
                    goto Succeed;
            }
            *dest-- = *pb--;
            --nb;
            if (nb == 1)
                goto CopyA;

            k = gallop_left(*pa, baseb, nb, nb-1, compare);
            if (k < 0)
                goto Fail;
            k = nb - k;
            bcount = k;
            if (k) {
                dest -= k;
                pb -= k;
                memcpy(dest+1, pb+1, k * sizeof(PyObject *));
                nb -= k;
                if (nb == 1)
                    goto CopyA;
                /* nb==0 is impossible now if the comparison
                 * function is consistent, but we can't assume
                 * that it is.
                 */
                if (nb == 0)
                    goto Succeed;
            }
            *dest-- = *pa--;
            --na;
            if (na == 0)
                goto Succeed;
        } while (acount >= MIN_GALLOP || bcount >= MIN_GALLOP);
        ++min_gallop;           /* penalize it for leaving galloping mode */
        ms->min_gallop = min_gallop;
    }
Succeed:
    result = 0;
Fail:
    if (nb)
        memcpy(dest-(nb-1), baseb, nb * sizeof(PyObject*));
    return result;
CopyA:
    assert(nb == 1 && na > 0);
    /* The first element of pb belongs at the front of the merge. */
    dest -= na;
    pa -= na;
    memmove(dest+1, pa+1, na * sizeof(PyObject *));
    *dest = *pb;
    return 0;
}

/* Merge the two runs at stack indices i and i+1.
 * Returns 0 on success, -1 on error.
 */
static Py_ssize_t
merge_at(MergeState *ms, Py_ssize_t i)
{
    PyObject **pa, **pb;
    Py_ssize_t na, nb;
    Py_ssize_t k;
    PyObject *compare;

    assert(ms != NULL);
    assert(ms->n >= 2);
    assert(i >= 0);
    assert(i == ms->n - 2 || i == ms->n - 3);

    pa = ms->pending[i].base;
    na = ms->pending[i].len;
    pb = ms->pending[i+1].base;
    nb = ms->pending[i+1].len;
    assert(na > 0 && nb > 0);
    assert(pa + na == pb);

    /* Record the length of the combined runs; if i is the 3rd-last
     * run now, also slide over the last run (which isn't involved
     * in this merge).  The current run i+1 goes away in any case.
     */
    ms->pending[i].len = na + nb;
    if (i == ms->n - 3)
        ms->pending[i+1] = ms->pending[i+2];
    --ms->n;

    /* Where does b start in a?  Elements in a before that can be
     * ignored (already in place).
     */
    compare = ms->compare;
    k = gallop_right(*pb, pa, na, 0, compare);
    if (k < 0)
        return -1;
    pa += k;
    na -= k;
    if (na == 0)
        return 0;

    /* Where does a end in b?  Elements in b after that can be
     * ignored (already in place).
     */
    nb = gallop_left(pa[na-1], pb, nb, nb-1, compare);
    if (nb <= 0)
        return nb;

    /* Merge what remains of the runs, using a temp array with
     * min(na, nb) elements.
     */
    if (na <= nb)
        return merge_lo(ms, pa, na, pb, nb);
    else
        return merge_hi(ms, pa, na, pb, nb);
}

/* Examine the stack of runs waiting to be merged, merging adjacent runs
 * until the stack invariants are re-established:
 *
 * 1. len[-3] > len[-2] + len[-1]
 * 2. len[-2] > len[-1]
 *
 * See listsort.txt for more info.
 *
 * Returns 0 on success, -1 on error.
 */
static int
merge_collapse(MergeState *ms)
{
    struct s_slice *p = ms->pending;

    assert(ms);
    while (ms->n > 1) {
        Py_ssize_t n = ms->n - 2;
        if ((n > 0 && p[n-1].len <= p[n].len + p[n+1].len) ||
            (n > 1 && p[n-2].len <= p[n-1].len + p[n].len)) {
            if (p[n-1].len < p[n+1].len)
                --n;
            if (merge_at(ms, n) < 0)
                return -1;
        }
        else if (p[n].len <= p[n+1].len) {
                 if (merge_at(ms, n) < 0)
                        return -1;
        }
        else
            break;
    }
    return 0;
}

/* Regardless of invariants, merge all runs on the stack until only one
 * remains.  This is used at the end of the mergesort.
 *
 * Returns 0 on success, -1 on error.
 */
static int
merge_force_collapse(MergeState *ms)
{
    struct s_slice *p = ms->pending;

    assert(ms);
    while (ms->n > 1) {
        Py_ssize_t n = ms->n - 2;
        if (n > 0 && p[n-1].len < p[n+1].len)
            --n;
        if (merge_at(ms, n) < 0)
            return -1;
    }
    return 0;
}

/* Compute a good value for the minimum run length; natural runs shorter
 * than this are boosted artificially via binary insertion.
 *
 * If n < 64, return n (it's too small to bother with fancy stuff).
 * Else if n is an exact power of 2, return 32.
 * Else return an int k, 32 <= k <= 64, such that n/k is close to, but
 * strictly less than, an exact power of 2.
 *
 * See listsort.txt for more info.
 */
static Py_ssize_t
merge_compute_minrun(Py_ssize_t n)
{
    Py_ssize_t r = 0;           /* becomes 1 if any 1 bits are shifted off */

    assert(n >= 0);
    while (n >= 64) {
        r |= n & 1;
        n >>= 1;
    }
    return n + r;
}

/* Special wrapper to support stable sorting using the decorate-sort-undecorate
   pattern.  Holds a key which is used for comparisons and the original record
   which is returned during the undecorate phase.  By exposing only the key
   during comparisons, the underlying sort stability characteristics are left
   unchanged.  Also, if a custom comparison function is used, it will only see
   the key instead of a full record. */

typedef struct {
    PyObject_HEAD
    PyObject *key;
    PyObject *value;
} sortwrapperobject;

PyDoc_STRVAR(sortwrapper_doc, "Object wrapper with a custom sort key.");
static PyObject *
sortwrapper_richcompare(sortwrapperobject *, sortwrapperobject *, int);
static void
sortwrapper_dealloc(sortwrapperobject *);

static PyTypeObject sortwrapper_type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "sortwrapper",                              /* tp_name */
    sizeof(sortwrapperobject),                  /* tp_basicsize */
    0,                                          /* tp_itemsize */
    /* methods */
    (destructor)sortwrapper_dealloc,            /* tp_dealloc */
    0,                                          /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_compare */
    0,                                          /* tp_repr */
    0,                                          /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    0,                                          /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT |
    Py_TPFLAGS_HAVE_RICHCOMPARE,                /* tp_flags */
    sortwrapper_doc,                            /* tp_doc */
    0,                                          /* tp_traverse */
    0,                                          /* tp_clear */
    (richcmpfunc)sortwrapper_richcompare,       /* tp_richcompare */
};


static PyObject *
sortwrapper_richcompare(sortwrapperobject *a, sortwrapperobject *b, int op)
{
    if (!PyObject_TypeCheck(b, &sortwrapper_type)) {
        PyErr_SetString(PyExc_TypeError,
            "expected a sortwrapperobject");
        return NULL;
    }
    return PyObject_RichCompare(a->key, b->key, op);
}

static void
sortwrapper_dealloc(sortwrapperobject *so)
{
    Py_XDECREF(so->key);
    Py_XDECREF(so->value);
    PyObject_Del(so);
}

/* Returns a new reference to a sortwrapper.
   Consumes the references to the two underlying objects. */

static PyObject *
build_sortwrapper(PyObject *key, PyObject *value)
{
    sortwrapperobject *so;

    so = PyObject_New(sortwrapperobject, &sortwrapper_type);
    if (so == NULL)
        return NULL;
    so->key = key;
    so->value = value;
    return (PyObject *)so;
}

/* Returns a new reference to the value underlying the wrapper. */
static PyObject *
sortwrapper_getvalue(PyObject *so)
{
    PyObject *value;

    if (!PyObject_TypeCheck(so, &sortwrapper_type)) {
        PyErr_SetString(PyExc_TypeError,
            "expected a sortwrapperobject");
        return NULL;
    }
    value = ((sortwrapperobject *)so)->value;
    Py_INCREF(value);
    return value;
}

/* Wrapper for user specified cmp functions in combination with a
   specified key function.  Makes sure the cmp function is presented
   with the actual key instead of the sortwrapper */

typedef struct {
    PyObject_HEAD
    PyObject *func;
} cmpwrapperobject;

static void
cmpwrapper_dealloc(cmpwrapperobject *co)
{
    Py_XDECREF(co->func);
    PyObject_Del(co);
}

static PyObject *
cmpwrapper_call(cmpwrapperobject *co, PyObject *args, PyObject *kwds)
{
    PyObject *x, *y, *xx, *yy;

    if (!PyArg_UnpackTuple(args, "", 2, 2, &x, &y))
        return NULL;
    if (!PyObject_TypeCheck(x, &sortwrapper_type) ||
        !PyObject_TypeCheck(y, &sortwrapper_type)) {
        PyErr_SetString(PyExc_TypeError,
            "expected a sortwrapperobject");
        return NULL;
    }
    xx = ((sortwrapperobject *)x)->key;
    yy = ((sortwrapperobject *)y)->key;
    return PyObject_CallFunctionObjArgs(co->func, xx, yy, NULL);
}

PyDoc_STRVAR(cmpwrapper_doc, "cmp() wrapper for sort with custom keys.");

static PyTypeObject cmpwrapper_type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "cmpwrapper",                               /* tp_name */
    sizeof(cmpwrapperobject),                   /* tp_basicsize */
    0,                                          /* tp_itemsize */
    /* methods */
    (destructor)cmpwrapper_dealloc,             /* tp_dealloc */
    0,                                          /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_compare */
    0,                                          /* tp_repr */
    0,                                          /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    0,                                          /* tp_hash */
    (ternaryfunc)cmpwrapper_call,               /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT,                         /* tp_flags */
    cmpwrapper_doc,                             /* tp_doc */
};

static PyObject *
build_cmpwrapper(PyObject *cmpfunc)
{
    cmpwrapperobject *co;

    co = PyObject_New(cmpwrapperobject, &cmpwrapper_type);
    if (co == NULL)
        return NULL;
    Py_INCREF(cmpfunc);
    co->func = cmpfunc;
    return (PyObject *)co;
}

/* An adaptive, stable, natural mergesort.  See listsort.txt.
 * Returns Py_None on success, NULL on error.  Even in case of error, the
 * list will be some permutation of its input state (nothing is lost or
 * duplicated).
 */
static PyObject *
listsort(PyListObject *self, PyObject *args, PyObject *kwds)
{
    MergeState ms;
    PyObject **lo, **hi;
    Py_ssize_t nremaining;
    Py_ssize_t minrun;
    Py_ssize_t saved_ob_size, saved_allocated;
    PyObject **saved_ob_item;
    PyObject **final_ob_item;
    PyObject *compare = NULL;
    PyObject *result = NULL;            /* guilty until proved innocent */
    int reverse = 0;
    PyObject *keyfunc = NULL;
    Py_ssize_t i;
    PyObject *key, *value, *kvpair;
    static char *kwlist[] = {"cmp", "key", "reverse", 0};

    assert(self != NULL);
    assert (PyList_Check(self));
    if (args != NULL) {
        if (!PyArg_ParseTupleAndKeywords(args, kwds, "|OOi:sort",
            kwlist, &compare, &keyfunc, &reverse))
            return NULL;
    }
    if (compare == Py_None)
        compare = NULL;
    if (compare != NULL &&
        PyErr_WarnPy3k("the cmp argument is not supported in 3.x", 1) < 0)
        return NULL;
    if (keyfunc == Py_None)
        keyfunc = NULL;
    if (compare != NULL && keyfunc != NULL) {
        compare = build_cmpwrapper(compare);
        if (compare == NULL)
            return NULL;
    } else
        Py_XINCREF(compare);

    /* The list is temporarily made empty, so that mutations performed
     * by comparison functions can't affect the slice of memory we're
     * sorting (allowing mutations during sorting is a core-dump
     * factory, since ob_item may change).
     */
    saved_ob_size = Py_SIZE(self);
    saved_ob_item = self->ob_item;
    saved_allocated = self->allocated;
    Py_SIZE(self) = 0;
    self->ob_item = NULL;
    self->allocated = -1; /* any operation will reset it to >= 0 */

    if (keyfunc != NULL) {
        for (i=0 ; i < saved_ob_size ; i++) {
            value = saved_ob_item[i];
            key = PyObject_CallFunctionObjArgs(keyfunc, value,
                                               NULL);
            if (key == NULL) {
                for (i=i-1 ; i>=0 ; i--) {
                    kvpair = saved_ob_item[i];
                    value = sortwrapper_getvalue(kvpair);
                    saved_ob_item[i] = value;
                    Py_DECREF(kvpair);
                }
                goto dsu_fail;
            }
            kvpair = build_sortwrapper(key, value);
            if (kvpair == NULL)
                goto dsu_fail;
            saved_ob_item[i] = kvpair;
        }
    }

    /* Reverse sort stability achieved by initially reversing the list,
    applying a stable forward sort, then reversing the final result. */
    if (reverse && saved_ob_size > 1)
        reverse_slice(saved_ob_item, saved_ob_item + saved_ob_size);

    merge_init(&ms, compare);

    nremaining = saved_ob_size;
    if (nremaining < 2)
        goto succeed;

    /* March over the array once, left to right, finding natural runs,
     * and extending short natural runs to minrun elements.
     */
    lo = saved_ob_item;
    hi = lo + nremaining;
    minrun = merge_compute_minrun(nremaining);
    do {
        int descending;
        Py_ssize_t n;

        /* Identify next run. */
        n = count_run(lo, hi, compare, &descending);
        if (n < 0)
            goto fail;
        if (descending)
            reverse_slice(lo, lo + n);
        /* If short, extend to min(minrun, nremaining). */
        if (n < minrun) {
            const Py_ssize_t force = nremaining <= minrun ?
                              nremaining : minrun;
            if (binarysort(lo, lo + force, lo + n, compare) < 0)
                goto fail;
            n = force;
        }
        /* Push run onto pending-runs stack, and maybe merge. */
        assert(ms.n < MAX_MERGE_PENDING);
        ms.pending[ms.n].base = lo;
        ms.pending[ms.n].len = n;
        ++ms.n;
        if (merge_collapse(&ms) < 0)
            goto fail;
        /* Advance to find next run. */
        lo += n;
        nremaining -= n;
    } while (nremaining);
    assert(lo == hi);

    if (merge_force_collapse(&ms) < 0)
        goto fail;
    assert(ms.n == 1);
    assert(ms.pending[0].base == saved_ob_item);
    assert(ms.pending[0].len == saved_ob_size);

succeed:
    result = Py_None;
fail:
    if (keyfunc != NULL) {
        for (i=0 ; i < saved_ob_size ; i++) {
            kvpair = saved_ob_item[i];
            value = sortwrapper_getvalue(kvpair);
            saved_ob_item[i] = value;
            Py_DECREF(kvpair);
        }
    }

    if (self->allocated != -1 && result != NULL) {
        /* The user mucked with the list during the sort,
         * and we don't already have another error to report.
         */
        PyErr_SetString(PyExc_ValueError, "list modified during sort");
        result = NULL;
    }

    if (reverse && saved_ob_size > 1)
        reverse_slice(saved_ob_item, saved_ob_item + saved_ob_size);

    merge_freemem(&ms);

dsu_fail:
    final_ob_item = self->ob_item;
    i = Py_SIZE(self);
    Py_SIZE(self) = saved_ob_size;
    self->ob_item = saved_ob_item;
    self->allocated = saved_allocated;
    if (final_ob_item != NULL) {
        /* we cannot use list_clear() for this because it does not
           guarantee that the list is really empty when it returns */
        while (--i >= 0) {
            Py_XDECREF(final_ob_item[i]);
        }
        PyMem_FREE(final_ob_item);
    }
    Py_XDECREF(compare);
    Py_XINCREF(result);
    return result;
}
#undef IFLT
#undef ISLT

int
PyList_Sort(PyObject *v)
{
    if (v == NULL || !PyList_Check(v)) {
        PyErr_BadInternalCall();
        return -1;
    }
    v = listsort((PyListObject *)v, (PyObject *)NULL, (PyObject *)NULL);
    if (v == NULL)
        return -1;
    Py_DECREF(v);
    return 0;
}

static PyObject *
listreverse(PyListObject *self)
{
    if (Py_SIZE(self) > 1)
        reverse_slice(self->ob_item, self->ob_item + Py_SIZE(self));
    Py_RETURN_NONE;
}

int
PyList_Reverse(PyObject *v)
{
    PyListObject *self = (PyListObject *)v;

    if (v == NULL || !PyList_Check(v)) {
        PyErr_BadInternalCall();
        return -1;
    }
    if (Py_SIZE(self) > 1)
        reverse_slice(self->ob_item, self->ob_item + Py_SIZE(self));
    return 0;
}

PyObject *
PyList_AsTuple(PyObject *v)
{
    PyObject *w;
    PyObject **p, **q;
    Py_ssize_t n;
    if (v == NULL || !PyList_Check(v)) {
        PyErr_BadInternalCall();
        return NULL;
    }
    n = Py_SIZE(v);
    w = PyTuple_New(n);
    if (w == NULL)
        return NULL;
    p = ((PyTupleObject *)w)->ob_item;
    q = ((PyListObject *)v)->ob_item;
    while (--n >= 0) {
        Py_INCREF(*q);
        *p = *q;
        p++;
        q++;
    }
    return w;
}

static PyObject *
listindex(PyListObject *self, PyObject *args)
{
    Py_ssize_t i, start=0, stop=Py_SIZE(self);
    PyObject *v, *format_tuple, *err_string;
    static PyObject *err_format = NULL;

    if (!PyArg_ParseTuple(args, "O|O&O&:index", &v,
                                _PyEval_SliceIndexNotNone, &start,
                                _PyEval_SliceIndexNotNone, &stop))
        return NULL;
    if (start < 0) {
        start += Py_SIZE(self);
        if (start < 0)
            start = 0;
    }
    if (stop < 0) {
        stop += Py_SIZE(self);
        if (stop < 0)
            stop = 0;
    }
    for (i = start; i < stop && i < Py_SIZE(self); i++) {
        int cmp = PyObject_RichCompareBool(self->ob_item[i], v, Py_EQ);
        if (cmp > 0)
            return PyInt_FromSsize_t(i);
        else if (cmp < 0)
            return NULL;
    }
    if (err_format == NULL) {
        err_format = PyString_FromString("%r is not in list");
        if (err_format == NULL)
            return NULL;
    }
    format_tuple = PyTuple_Pack(1, v);
    if (format_tuple == NULL)
        return NULL;
    err_string = PyString_Format(err_format, format_tuple);
    Py_DECREF(format_tuple);
    if (err_string == NULL)
        return NULL;
    PyErr_SetObject(PyExc_ValueError, err_string);
    Py_DECREF(err_string);
    return NULL;
}

static PyObject *
listcount(PyListObject *self, PyObject *v)
{
    Py_ssize_t count = 0;
    Py_ssize_t i;

    for (i = 0; i < Py_SIZE(self); i++) {
        int cmp = PyObject_RichCompareBool(self->ob_item[i], v, Py_EQ);
        if (cmp > 0)
            count++;
        else if (cmp < 0)
            return NULL;
    }
    return PyInt_FromSsize_t(count);
}

static PyObject *
listremove(PyListObject *self, PyObject *v)
{
    Py_ssize_t i;

    for (i = 0; i < Py_SIZE(self); i++) {
        int cmp = PyObject_RichCompareBool(self->ob_item[i], v, Py_EQ);
        if (cmp > 0) {
            if (list_ass_slice(self, i, i+1,
                               (PyObject *)NULL) == 0)
                Py_RETURN_NONE;
            return NULL;
        }
        else if (cmp < 0)
            return NULL;
    }
    PyErr_SetString(PyExc_ValueError, "list.remove(x): x not in list");
    return NULL;
}

static int
list_traverse(PyListObject *o, visitproc visit, void *arg)
{
    Py_ssize_t i;

    for (i = Py_SIZE(o); --i >= 0; )
        Py_VISIT(o->ob_item[i]);
    return 0;
}

static PyObject *
list_richcompare(PyObject *v, PyObject *w, int op)
{
    PyListObject *vl, *wl;
    Py_ssize_t i;

    if (!PyList_Check(v) || !PyList_Check(w)) {
        Py_INCREF(Py_NotImplemented);
        return Py_NotImplemented;
    }

    vl = (PyListObject *)v;
    wl = (PyListObject *)w;

    if (Py_SIZE(vl) != Py_SIZE(wl) && (op == Py_EQ || op == Py_NE)) {
        /* Shortcut: if the lengths differ, the lists differ */
        PyObject *res;
        if (op == Py_EQ)
            res = Py_False;
        else
            res = Py_True;
        Py_INCREF(res);
        return res;
    }

    /* Search for the first index where items are different */
    for (i = 0; i < Py_SIZE(vl) && i < Py_SIZE(wl); i++) {
        int k = PyObject_RichCompareBool(vl->ob_item[i],
                                         wl->ob_item[i], Py_EQ);
        if (k < 0)
            return NULL;
        if (!k)
            break;
    }

    if (i >= Py_SIZE(vl) || i >= Py_SIZE(wl)) {
        /* No more items to compare -- compare sizes */
        Py_ssize_t vs = Py_SIZE(vl);
        Py_ssize_t ws = Py_SIZE(wl);
        int cmp;
        PyObject *res;
        switch (op) {
        case Py_LT: cmp = vs <  ws; break;
        case Py_LE: cmp = vs <= ws; break;
        case Py_EQ: cmp = vs == ws; break;
        case Py_NE: cmp = vs != ws; break;
        case Py_GT: cmp = vs >  ws; break;
        case Py_GE: cmp = vs >= ws; break;
        default: return NULL; /* cannot happen */
        }
        if (cmp)
            res = Py_True;
        else
            res = Py_False;
        Py_INCREF(res);
        return res;
    }

    /* We have an item that differs -- shortcuts for EQ/NE */
    if (op == Py_EQ) {
        Py_INCREF(Py_False);
        return Py_False;
    }
    if (op == Py_NE) {
        Py_INCREF(Py_True);
        return Py_True;
    }

    /* Compare the final item again using the proper operator */
    return PyObject_RichCompare(vl->ob_item[i], wl->ob_item[i], op);
}

static int
list_init(PyListObject *self, PyObject *args, PyObject *kw)
{
    PyObject *arg = NULL;
    static char *kwlist[] = {"sequence", 0};

    if (!PyArg_ParseTupleAndKeywords(args, kw, "|O:list", kwlist, &arg))
        return -1;

    /* Verify list invariants established by PyType_GenericAlloc() */
    assert(0 <= Py_SIZE(self));
    assert(Py_SIZE(self) <= self->allocated || self->allocated == -1);
    assert(self->ob_item != NULL ||
           self->allocated == 0 || self->allocated == -1);

    /* Empty previous contents */
    if (self->ob_item != NULL) {
        (void)list_clear(self);
    }
    if (arg != NULL) {
        PyObject *rv = listextend(self, arg);
        if (rv == NULL)
            return -1;
        Py_DECREF(rv);
    }
    return 0;
}

static PyObject *
list_sizeof(PyListObject *self)
{
    Py_ssize_t res;

    res = _PyObject_SIZE(Py_TYPE(self)) + self->allocated * sizeof(void*);
    return PyInt_FromSsize_t(res);
}

static PyObject *list_iter(PyObject *seq);
static PyObject *list_reversed(PyListObject* seq, PyObject* unused);

PyDoc_STRVAR(getitem_doc,
"x.__getitem__(y) <==> x[y]");
PyDoc_STRVAR(reversed_doc,
"L.__reversed__() -- return a reverse iterator over the list");
PyDoc_STRVAR(sizeof_doc,
"L.__sizeof__() -- size of L in memory, in bytes");
PyDoc_STRVAR(append_doc,
"L.append(object) -- append object to end");
PyDoc_STRVAR(extend_doc,
"L.extend(iterable) -- extend list by appending elements from the iterable");
PyDoc_STRVAR(insert_doc,
"L.insert(index, object) -- insert object before index");
PyDoc_STRVAR(pop_doc,
"L.pop([index]) -> item -- remove and return item at index (default last).\n"
"Raises IndexError if list is empty or index is out of range.");
PyDoc_STRVAR(remove_doc,
"L.remove(value) -- remove first occurrence of value.\n"
"Raises ValueError if the value is not present.");
PyDoc_STRVAR(index_doc,
"L.index(value, [start, [stop]]) -> integer -- return first index of value.\n"
"Raises ValueError if the value is not present.");
PyDoc_STRVAR(count_doc,
"L.count(value) -> integer -- return number of occurrences of value");
PyDoc_STRVAR(reverse_doc,
"L.reverse() -- reverse *IN PLACE*");
PyDoc_STRVAR(sort_doc,
"L.sort(cmp=None, key=None, reverse=False) -- stable sort *IN PLACE*;\n\
cmp(x, y) -> -1, 0, 1");

static PyObject *list_subscript(PyListObject*, PyObject*);

static PyMethodDef list_methods[] = {
    {"__getitem__", (PyCFunction)list_subscript, METH_O|METH_COEXIST, getitem_doc},
    {"__reversed__",(PyCFunction)list_reversed, METH_NOARGS, reversed_doc},
    {"__sizeof__",  (PyCFunction)list_sizeof, METH_NOARGS, sizeof_doc},
    {"append",          (PyCFunction)listappend,  METH_O, append_doc},
    {"insert",          (PyCFunction)listinsert,  METH_VARARGS, insert_doc},
    {"extend",      (PyCFunction)listextend,  METH_O, extend_doc},
    {"pop",             (PyCFunction)listpop,     METH_VARARGS, pop_doc},
    {"remove",          (PyCFunction)listremove,  METH_O, remove_doc},
    {"index",           (PyCFunction)listindex,   METH_VARARGS, index_doc},
    {"count",           (PyCFunction)listcount,   METH_O, count_doc},
    {"reverse",         (PyCFunction)listreverse, METH_NOARGS, reverse_doc},
    {"sort",            (PyCFunction)listsort,    METH_VARARGS | METH_KEYWORDS, sort_doc},
    {NULL,              NULL}           /* sentinel */
};

static PySequenceMethods list_as_sequence = {
    (lenfunc)list_length,                       /* sq_length */
    (binaryfunc)list_concat,                    /* sq_concat */
    (ssizeargfunc)list_repeat,                  /* sq_repeat */
    (ssizeargfunc)list_item,                    /* sq_item */
    (ssizessizeargfunc)list_slice,              /* sq_slice */
    (ssizeobjargproc)list_ass_item,             /* sq_ass_item */
    (ssizessizeobjargproc)list_ass_slice,       /* sq_ass_slice */
    (objobjproc)list_contains,                  /* sq_contains */
    (binaryfunc)list_inplace_concat,            /* sq_inplace_concat */
    (ssizeargfunc)list_inplace_repeat,          /* sq_inplace_repeat */
};

PyDoc_STRVAR(list_doc,
"list() -> new empty list\n"
"list(iterable) -> new list initialized from iterable's items");


static PyObject *
list_subscript(PyListObject* self, PyObject* item)
{
    if (PyIndex_Check(item)) {
        Py_ssize_t i;
        i = PyNumber_AsSsize_t(item, PyExc_IndexError);
        if (i == -1 && PyErr_Occurred())
            return NULL;
        if (i < 0)
            i += PyList_GET_SIZE(self);
        return list_item(self, i);
    }
    else if (PySlice_Check(item)) {
        Py_ssize_t start, stop, step, slicelength, cur, i;
        PyObject* result;
        PyObject* it;
        PyObject **src, **dest;

        if (_PySlice_Unpack(item, &start, &stop, &step) < 0) {
            return NULL;
        }
        slicelength = _PySlice_AdjustIndices(Py_SIZE(self), &start, &stop,
                                            step);

        if (slicelength <= 0) {
            return PyList_New(0);
        }
        else if (step == 1) {
            return list_slice(self, start, stop);
        }
        else {
            result = PyList_New(slicelength);
            if (!result) return NULL;

            src = self->ob_item;
            dest = ((PyListObject *)result)->ob_item;
            for (cur = start, i = 0; i < slicelength;
                 cur += step, i++) {
                it = src[cur];
                Py_INCREF(it);
                dest[i] = it;
            }

            return result;
        }
    }
    else {
        PyErr_Format(PyExc_TypeError,
                     "list indices must be integers, not %.200s",
                     item->ob_type->tp_name);
        return NULL;
    }
}

static int
list_ass_subscript(PyListObject* self, PyObject* item, PyObject* value)
{
    if (PyIndex_Check(item)) {
        Py_ssize_t i = PyNumber_AsSsize_t(item, PyExc_IndexError);
        if (i == -1 && PyErr_Occurred())
            return -1;
        if (i < 0)
            i += PyList_GET_SIZE(self);
        return list_ass_item(self, i, value);
    }
    else if (PySlice_Check(item)) {
        Py_ssize_t start, stop, step, slicelength;

        if (_PySlice_Unpack(item, &start, &stop, &step) < 0) {
            return -1;
        }
        slicelength = _PySlice_AdjustIndices(Py_SIZE(self), &start, &stop,
                                            step);

        if (step == 1)
            return list_ass_slice(self, start, stop, value);

        /* Make sure s[5:2] = [..] inserts at the right place:
           before 5, not before 2. */
        if ((step < 0 && start < stop) ||
            (step > 0 && start > stop))
            stop = start;

        if (value == NULL) {
            /* delete slice */
            PyObject **garbage;
            size_t cur;
            Py_ssize_t i;

            if (slicelength <= 0)
                return 0;

            if (step < 0) {
                stop = start + 1;
                start = stop + step*(slicelength - 1) - 1;
                step = -step;
            }

            assert((size_t)slicelength <=
                   PY_SIZE_MAX / sizeof(PyObject*));

            garbage = (PyObject**)
                PyMem_MALLOC(slicelength*sizeof(PyObject*));
            if (!garbage) {
                PyErr_NoMemory();
                return -1;
            }

            /* drawing pictures might help understand these for
               loops. Basically, we memmove the parts of the
               list that are *not* part of the slice: step-1
               items for each item that is part of the slice,
               and then tail end of the list that was not
               covered by the slice */
            for (cur = start, i = 0;
                 cur < (size_t)stop;
                 cur += step, i++) {
                Py_ssize_t lim = step - 1;

                garbage[i] = PyList_GET_ITEM(self, cur);

                if (cur + step >= (size_t)Py_SIZE(self)) {
                    lim = Py_SIZE(self) - cur - 1;
                }

                memmove(self->ob_item + cur - i,
                    self->ob_item + cur + 1,
                    lim * sizeof(PyObject *));
            }
            cur = start + slicelength*step;
            if (cur < (size_t)Py_SIZE(self)) {
                memmove(self->ob_item + cur - slicelength,
                    self->ob_item + cur,
                    (Py_SIZE(self) - cur) *
                     sizeof(PyObject *));
            }

            Py_SIZE(self) -= slicelength;
            list_resize(self, Py_SIZE(self));

            for (i = 0; i < slicelength; i++) {
                Py_DECREF(garbage[i]);
            }
            PyMem_FREE(garbage);

            return 0;
        }
        else {
            /* assign slice */
            PyObject *ins, *seq;
            PyObject **garbage, **seqitems, **selfitems;
            Py_ssize_t cur, i;

            /* protect against a[::-1] = a */
            if (self == (PyListObject*)value) {
                seq = list_slice((PyListObject*)value, 0,
                                   PyList_GET_SIZE(value));
            }
            else {
                seq = PySequence_Fast(value,
                                      "must assign iterable "
                                      "to extended slice");
            }
            if (!seq)
                return -1;

            if (PySequence_Fast_GET_SIZE(seq) != slicelength) {
                PyErr_Format(PyExc_ValueError,
                    "attempt to assign sequence of "
                    "size %zd to extended slice of "
                    "size %zd",
                         PySequence_Fast_GET_SIZE(seq),
                         slicelength);
                Py_DECREF(seq);
                return -1;
            }

            if (!slicelength) {
                Py_DECREF(seq);
                return 0;
            }

            garbage = (PyObject**)
                PyMem_MALLOC(slicelength*sizeof(PyObject*));
            if (!garbage) {
                Py_DECREF(seq);
                PyErr_NoMemory();
                return -1;
            }

            selfitems = self->ob_item;
            seqitems = PySequence_Fast_ITEMS(seq);
            for (cur = start, i = 0; i < slicelength;
                 cur += step, i++) {
                garbage[i] = selfitems[cur];
                ins = seqitems[i];
                Py_INCREF(ins);
                selfitems[cur] = ins;
            }

            for (i = 0; i < slicelength; i++) {
                Py_DECREF(garbage[i]);
            }

            PyMem_FREE(garbage);
            Py_DECREF(seq);

            return 0;
        }
    }
    else {
        PyErr_Format(PyExc_TypeError,
                     "list indices must be integers, not %.200s",
                     item->ob_type->tp_name);
        return -1;
    }
}

static PyMappingMethods list_as_mapping = {
    (lenfunc)list_length,
    (binaryfunc)list_subscript,
    (objobjargproc)list_ass_subscript
};

PyTypeObject PyList_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "list",
    sizeof(PyListObject),
    0,
    (destructor)list_dealloc,                   /* tp_dealloc */
    (printfunc)list_print,                      /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_compare */
    (reprfunc)list_repr,                        /* tp_repr */
    0,                                          /* tp_as_number */
    &list_as_sequence,                          /* tp_as_sequence */
    &list_as_mapping,                           /* tp_as_mapping */
    (hashfunc)PyObject_HashNotImplemented,      /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
        Py_TPFLAGS_BASETYPE | Py_TPFLAGS_LIST_SUBCLASS,         /* tp_flags */
    list_doc,                                   /* tp_doc */
    (traverseproc)list_traverse,                /* tp_traverse */
    (inquiry)list_clear,                        /* tp_clear */
    list_richcompare,                           /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    list_iter,                                  /* tp_iter */
    0,                                          /* tp_iternext */
    list_methods,                               /* tp_methods */
    0,                                          /* tp_members */
    0,                                          /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    0,                                          /* tp_descr_get */
    0,                                          /* tp_descr_set */
    0,                                          /* tp_dictoffset */
    (initproc)list_init,                        /* tp_init */
    PyType_GenericAlloc,                        /* tp_alloc */
    PyType_GenericNew,                          /* tp_new */
    PyObject_GC_Del,                            /* tp_free */
};


/*********************** List Iterator **************************/

typedef struct {
    PyObject_HEAD
    long it_index;
    PyListObject *it_seq; /* Set to NULL when iterator is exhausted */
} listiterobject;

static PyObject *list_iter(PyObject *);
static void listiter_dealloc(listiterobject *);
static int listiter_traverse(listiterobject *, visitproc, void *);
static PyObject *listiter_next(listiterobject *);
static PyObject *listiter_len(listiterobject *);

PyDoc_STRVAR(length_hint_doc, "Private method returning an estimate of len(list(it)).");

static PyMethodDef listiter_methods[] = {
    {"__length_hint__", (PyCFunction)listiter_len, METH_NOARGS, length_hint_doc},
    {NULL,              NULL}           /* sentinel */
};

PyTypeObject PyListIter_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "listiterator",                             /* tp_name */
    sizeof(listiterobject),                     /* tp_basicsize */
    0,                                          /* tp_itemsize */
    /* methods */
    (destructor)listiter_dealloc,               /* tp_dealloc */
    0,                                          /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_compare */
    0,                                          /* tp_repr */
    0,                                          /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    0,                                          /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC,/* tp_flags */
    0,                                          /* tp_doc */
    (traverseproc)listiter_traverse,            /* tp_traverse */
    0,                                          /* tp_clear */
    0,                                          /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    PyObject_SelfIter,                          /* tp_iter */
    (iternextfunc)listiter_next,                /* tp_iternext */
    listiter_methods,                           /* tp_methods */
    0,                                          /* tp_members */
};


static PyObject *
list_iter(PyObject *seq)
{
    listiterobject *it;

    if (!PyList_Check(seq)) {
        PyErr_BadInternalCall();
        return NULL;
    }
    it = PyObject_GC_New(listiterobject, &PyListIter_Type);
    if (it == NULL)
        return NULL;
    it->it_index = 0;
    Py_INCREF(seq);
    it->it_seq = (PyListObject *)seq;
    _PyObject_GC_TRACK(it);
    return (PyObject *)it;
}

static void
listiter_dealloc(listiterobject *it)
{
    _PyObject_GC_UNTRACK(it);
    Py_XDECREF(it->it_seq);
    PyObject_GC_Del(it);
}

static int
listiter_traverse(listiterobject *it, visitproc visit, void *arg)
{
    Py_VISIT(it->it_seq);
    return 0;
}

static PyObject *
listiter_next(listiterobject *it)
{
    PyListObject *seq;
    PyObject *item;

    assert(it != NULL);
    seq = it->it_seq;
    if (seq == NULL)
        return NULL;
    assert(PyList_Check(seq));

    if (it->it_index < PyList_GET_SIZE(seq)) {
        item = PyList_GET_ITEM(seq, it->it_index);
        ++it->it_index;
        Py_INCREF(item);
        return item;
    }

    it->it_seq = NULL;
    Py_DECREF(seq);
    return NULL;
}

static PyObject *
listiter_len(listiterobject *it)
{
    Py_ssize_t len;
    if (it->it_seq) {
        len = PyList_GET_SIZE(it->it_seq) - it->it_index;
        if (len >= 0)
            return PyInt_FromSsize_t(len);
    }
    return PyInt_FromLong(0);
}
/*********************** List Reverse Iterator **************************/

typedef struct {
    PyObject_HEAD
    Py_ssize_t it_index;
    PyListObject *it_seq; /* Set to NULL when iterator is exhausted */
} listreviterobject;

static PyObject *list_reversed(PyListObject *, PyObject *);
static void listreviter_dealloc(listreviterobject *);
static int listreviter_traverse(listreviterobject *, visitproc, void *);
static PyObject *listreviter_next(listreviterobject *);
static PyObject *listreviter_len(listreviterobject *);

static PyMethodDef listreviter_methods[] = {
    {"__length_hint__", (PyCFunction)listreviter_len, METH_NOARGS, length_hint_doc},
    {NULL,              NULL}           /* sentinel */
};

PyTypeObject PyListRevIter_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "listreverseiterator",                      /* tp_name */
    sizeof(listreviterobject),                  /* tp_basicsize */
    0,                                          /* tp_itemsize */
    /* methods */
    (destructor)listreviter_dealloc,            /* tp_dealloc */
    0,                                          /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_compare */
    0,                                          /* tp_repr */
    0,                                          /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    0,                                          /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC,/* tp_flags */
    0,                                          /* tp_doc */
    (traverseproc)listreviter_traverse,         /* tp_traverse */
    0,                                          /* tp_clear */
    0,                                          /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    PyObject_SelfIter,                          /* tp_iter */
    (iternextfunc)listreviter_next,             /* tp_iternext */
    listreviter_methods,                /* tp_methods */
    0,
};

static PyObject *
list_reversed(PyListObject *seq, PyObject *unused)
{
    listreviterobject *it;

    it = PyObject_GC_New(listreviterobject, &PyListRevIter_Type);
    if (it == NULL)
        return NULL;
    assert(PyList_Check(seq));
    it->it_index = PyList_GET_SIZE(seq) - 1;
    Py_INCREF(seq);
    it->it_seq = seq;
    PyObject_GC_Track(it);
    return (PyObject *)it;
}

static void
listreviter_dealloc(listreviterobject *it)
{
    PyObject_GC_UnTrack(it);
    Py_XDECREF(it->it_seq);
    PyObject_GC_Del(it);
}

static int
listreviter_traverse(listreviterobject *it, visitproc visit, void *arg)
{
    Py_VISIT(it->it_seq);
    return 0;
}

static PyObject *
listreviter_next(listreviterobject *it)
{
    PyObject *item;
    Py_ssize_t index;
    PyListObject *seq;

    assert(it != NULL);
    seq = it->it_seq;
    if (seq == NULL) {
        return NULL;
    }
    assert(PyList_Check(seq));

    index = it->it_index;
    if (index>=0 && index < PyList_GET_SIZE(seq)) {
        item = PyList_GET_ITEM(seq, index);
        it->it_index--;
        Py_INCREF(item);
        return item;
    }
    it->it_index = -1;
    it->it_seq = NULL;
    Py_DECREF(seq);
    return NULL;
}

static PyObject *
listreviter_len(listreviterobject *it)
{
    Py_ssize_t len = it->it_index + 1;
    if (it->it_seq == NULL || PyList_GET_SIZE(it->it_seq) < len)
        len = 0;
    return PyLong_FromSsize_t(len);
}

```

### Object/listsort.txt

```
Intro
-----
This describes an adaptive, stable, natural mergesort, modestly called
timsort (hey, I earned it <wink>).  It has supernatural performance on many
kinds of partially ordered arrays (less than lg(N!) comparisons needed, and
as few as N-1), yet as fast as Python's previous highly tuned samplesort
hybrid on random arrays.

In a nutshell, the main routine marches over the array once, left to right,
alternately identifying the next run, then merging it into the previous
runs "intelligently".  Everything else is complication for speed, and some
hard-won measure of memory efficiency.


Comparison with Python's Samplesort Hybrid
------------------------------------------
+ timsort can require a temp array containing as many as N//2 pointers,
  which means as many as 2*N extra bytes on 32-bit boxes.  It can be
  expected to require a temp array this large when sorting random data; on
  data with significant structure, it may get away without using any extra
  heap memory.  This appears to be the strongest argument against it, but
  compared to the size of an object, 2 temp bytes worst-case (also expected-
  case for random data) doesn't scare me much.

  It turns out that Perl is moving to a stable mergesort, and the code for
  that appears always to require a temp array with room for at least N
  pointers. (Note that I wouldn't want to do that even if space weren't an
  issue; I believe its efforts at memory frugality also save timsort
  significant pointer-copying costs, and allow it to have a smaller working
  set.)

+ Across about four hours of generating random arrays, and sorting them
  under both methods, samplesort required about 1.5% more comparisons
  (the program is at the end of this file).

+ In real life, this may be faster or slower on random arrays than
  samplesort was, depending on platform quirks.  Since it does fewer
  comparisons on average, it can be expected to do better the more
  expensive a comparison function is.  OTOH, it does more data movement
  (pointer copying) than samplesort, and that may negate its small
  comparison advantage (depending on platform quirks) unless comparison
  is very expensive.

+ On arrays with many kinds of pre-existing order, this blows samplesort out
  of the water.  It's significantly faster than samplesort even on some
  cases samplesort was special-casing the snot out of.  I believe that lists
  very often do have exploitable partial order in real life, and this is the
  strongest argument in favor of timsort (indeed, samplesort's special cases
  for extreme partial order are appreciated by real users, and timsort goes
  much deeper than those, in particular naturally covering every case where
  someone has suggested "and it would be cool if list.sort() had a special
  case for this too ... and for that ...").

+ Here are exact comparison counts across all the tests in sortperf.py,
  when run with arguments "15 20 1".

  Column Key:
      *sort: random data
      \sort: descending data
      /sort: ascending data
      3sort: ascending, then 3 random exchanges
      +sort: ascending, then 10 random at the end
      %sort: ascending, then randomly replace 1% of elements w/ random values
      ~sort: many duplicates
      =sort: all equal
      !sort: worst case scenario

  First the trivial cases, trivial for samplesort because it special-cased
  them, and trivial for timsort because it naturally works on runs.  Within
  an "n" block, the first line gives the # of compares done by samplesort,
  the second line by timsort, and the third line is the percentage by
  which the samplesort count exceeds the timsort count:

      n   \sort   /sort   =sort
-------  ------  ------  ------
  32768   32768   32767   32767  samplesort
          32767   32767   32767  timsort
          0.00%   0.00%   0.00%  (samplesort - timsort) / timsort

  65536   65536   65535   65535
          65535   65535   65535
          0.00%   0.00%   0.00%

 131072  131072  131071  131071
         131071  131071  131071
          0.00%   0.00%   0.00%

 262144  262144  262143  262143
         262143  262143  262143
          0.00%   0.00%   0.00%

 524288  524288  524287  524287
         524287  524287  524287
          0.00%   0.00%   0.00%

1048576 1048576 1048575 1048575
        1048575 1048575 1048575
          0.00%   0.00%   0.00%

  The algorithms are effectively identical in these cases, except that
  timsort does one less compare in \sort.

  Now for the more interesting cases.  Where lg(x) is the logarithm of x to
  the base 2 (e.g., lg(8)=3), lg(n!) is the information-theoretic limit for
  the best any comparison-based sorting algorithm can do on average (across
  all permutations).  When a method gets significantly below that, it's
  either astronomically lucky, or is finding exploitable structure in the
  data.


      n   lg(n!)    *sort    3sort     +sort   %sort    ~sort     !sort
-------  -------   ------   -------  -------  ------  -------  --------
  32768   444255   453096   453614    32908   452871   130491    469141 old
                   448885    33016    33007    50426   182083     65534 new
                    0.94% 1273.92%   -0.30%  798.09%  -28.33%   615.87% %ch from new

  65536   954037   972699   981940    65686   973104   260029   1004607
                   962991    65821    65808   101667   364341    131070
                    1.01% 1391.83%   -0.19%  857.15%  -28.63%   666.47%

 131072  2039137  2101881  2091491   131232  2092894   554790   2161379
                  2057533   131410   131361   206193   728871    262142
                    2.16% 1491.58%   -0.10%  915.02%  -23.88%   724.51%

 262144  4340409  4464460  4403233   262314  4445884  1107842   4584560
                  4377402   262437   262459   416347  1457945    524286
                    1.99% 1577.82%   -0.06%  967.83%  -24.01%   774.44%

 524288  9205096  9453356  9408463   524468  9441930  2218577   9692015
                  9278734   524580   524633   837947  2916107   1048574
                   1.88%  1693.52%   -0.03% 1026.79%  -23.92%   824.30%

1048576 19458756 19950272 19838588  1048766 19912134  4430649  20434212
                 19606028  1048958  1048941  1694896  5832445   2097150
                    1.76% 1791.27%   -0.02% 1074.83%  -24.03%   874.38%

  Discussion of cases:

  *sort:  There's no structure in random data to exploit, so the theoretical
  limit is lg(n!).  Both methods get close to that, and timsort is hugging
  it (indeed, in a *marginal* sense, it's a spectacular improvement --
  there's only about 1% left before hitting the wall, and timsort knows
  darned well it's doing compares that won't pay on random data -- but so
  does the samplesort hybrid).  For contrast, Hoare's original random-pivot
  quicksort does about 39% more compares than the limit, and the median-of-3
  variant about 19% more.

  3sort, %sort, and !sort:  No contest; there's structure in this data, but
  not of the specific kinds samplesort special-cases.  Note that structure
  in !sort wasn't put there on purpose -- it was crafted as a worst case for
  a previous quicksort implementation.  That timsort nails it came as a
  surprise to me (although it's obvious in retrospect).

  +sort:  samplesort special-cases this data, and does a few less compares
  than timsort.  However, timsort runs this case significantly faster on all
  boxes we have timings for, because timsort is in the business of merging
  runs efficiently, while samplesort does much more data movement in this
  (for it) special case.

  ~sort:  samplesort's special cases for large masses of equal elements are
  extremely effective on ~sort's specific data pattern, and timsort just
  isn't going to get close to that, despite that it's clearly getting a
  great deal of benefit out of the duplicates (the # of compares is much less
  than lg(n!)).  ~sort has a perfectly uniform distribution of just 4
  distinct values, and as the distribution gets more skewed, samplesort's
  equal-element gimmicks become less effective, while timsort's adaptive
  strategies find more to exploit; in a database supplied by Kevin Altis, a
  sort on its highly skewed "on which stock exchange does this company's
  stock trade?" field ran over twice as fast under timsort.

  However, despite that timsort does many more comparisons on ~sort, and
  that on several platforms ~sort runs highly significantly slower under
  timsort, on other platforms ~sort runs highly significantly faster under
  timsort.  No other kind of data has shown this wild x-platform behavior,
  and we don't have an explanation for it.  The only thing I can think of
  that could transform what "should be" highly significant slowdowns into
  highly significant speedups on some boxes are catastrophic cache effects
  in samplesort.

  But timsort "should be" slower than samplesort on ~sort, so it's hard
  to count that it isn't on some boxes as a strike against it <wink>.

+ Here's the highwater mark for the number of heap-based temp slots (4
  bytes each on this box) needed by each test, again with arguments
  "15 20 1":

   2**i  *sort \sort /sort  3sort  +sort  %sort  ~sort  =sort  !sort
  32768  16384     0     0   6256      0  10821  12288      0  16383
  65536  32766     0     0  21652      0  31276  24576      0  32767
 131072  65534     0     0  17258      0  58112  49152      0  65535
 262144 131072     0     0  35660      0 123561  98304      0 131071
 524288 262142     0     0  31302      0 212057 196608      0 262143
1048576 524286     0     0 312438      0 484942 393216      0 524287

  Discussion:  The tests that end up doing (close to) perfectly balanced
  merges (*sort, !sort) need all N//2 temp slots (or almost all).  ~sort
  also ends up doing balanced merges, but systematically benefits a lot from
  the preliminary pre-merge searches described under "Merge Memory" later.
  %sort approaches having a balanced merge at the end because the random
  selection of elements to replace is expected to produce an out-of-order
  element near the midpoint.  \sort, /sort, =sort are the trivial one-run
  cases, needing no merging at all.  +sort ends up having one very long run
  and one very short, and so gets all the temp space it needs from the small
  temparray member of the MergeState struct (note that the same would be
  true if the new random elements were prefixed to the sorted list instead,
  but not if they appeared "in the middle").  3sort approaches N//3 temp
  slots twice, but the run lengths that remain after 3 random exchanges
  clearly has very high variance.


A detailed description of timsort follows.

Runs
----
count_run() returns the # of elements in the next run.  A run is either
"ascending", which means non-decreasing:

    a0 <= a1 <= a2 <= ...

or "descending", which means strictly decreasing:

    a0 > a1 > a2 > ...

Note that a run is always at least 2 long, unless we start at the array's
last element.

The definition of descending is strict, because the main routine reverses
a descending run in-place, transforming a descending run into an ascending
run.  Reversal is done via the obvious fast "swap elements starting at each
end, and converge at the middle" method, and that can violate stability if
the slice contains any equal elements.  Using a strict definition of
descending ensures that a descending run contains distinct elements.

If an array is random, it's very unlikely we'll see long runs.  If a natural
run contains less than minrun elements (see next section), the main loop
artificially boosts it to minrun elements, via a stable binary insertion sort
applied to the right number of array elements following the short natural
run.  In a random array, *all* runs are likely to be minrun long as a
result.  This has two primary good effects:

1. Random data strongly tends then toward perfectly balanced (both runs have
   the same length) merges, which is the most efficient way to proceed when
   data is random.

2. Because runs are never very short, the rest of the code doesn't make
   heroic efforts to shave a few cycles off per-merge overheads.  For
   example, reasonable use of function calls is made, rather than trying to
   inline everything.  Since there are no more than N/minrun runs to begin
   with, a few "extra" function calls per merge is barely measurable.


Computing minrun
----------------
If N < 64, minrun is N.  IOW, binary insertion sort is used for the whole
array then; it's hard to beat that given the overheads of trying something
fancier (see note BINSORT).

When N is a power of 2, testing on random data showed that minrun values of
16, 32, 64 and 128 worked about equally well.  At 256 the data-movement cost
in binary insertion sort clearly hurt, and at 8 the increase in the number
of function calls clearly hurt.  Picking *some* power of 2 is important
here, so that the merges end up perfectly balanced (see next section).  We
pick 32 as a good value in the sweet range; picking a value at the low end
allows the adaptive gimmicks more opportunity to exploit shorter natural
runs.

Because sortperf.py only tries powers of 2, it took a long time to notice
that 32 isn't a good choice for the general case!  Consider N=2112:

>>> divmod(2112, 32)
(66, 0)
>>>

If the data is randomly ordered, we're very likely to end up with 66 runs
each of length 32.  The first 64 of these trigger a sequence of perfectly
balanced merges (see next section), leaving runs of lengths 2048 and 64 to
merge at the end.  The adaptive gimmicks can do that with fewer than 2048+64
compares, but it's still more compares than necessary, and-- mergesort's
bugaboo relative to samplesort --a lot more data movement (O(N) copies just
to get 64 elements into place).

If we take minrun=33 in this case, then we're very likely to end up with 64
runs each of length 33, and then all merges are perfectly balanced.  Better!

What we want to avoid is picking minrun such that in

    q, r = divmod(N, minrun)

q is a power of 2 and r>0 (then the last merge only gets r elements into
place, and r < minrun is small compared to N), or q a little larger than a
power of 2 regardless of r (then we've got a case similar to "2112", again
leaving too little work for the last merge to do).

Instead we pick a minrun in range(32, 65) such that N/minrun is exactly a
power of 2, or if that isn't possible, is close to, but strictly less than,
a power of 2.  This is easier to do than it may sound:  take the first 6
bits of N, and add 1 if any of the remaining bits are set.  In fact, that
rule covers every case in this section, including small N and exact powers
of 2; merge_compute_minrun() is a deceptively simple function.


The Merge Pattern
-----------------
In order to exploit regularities in the data, we're merging on natural
run lengths, and they can become wildly unbalanced.  That's a Good Thing
for this sort!  It means we have to find a way to manage an assortment of
potentially very different run lengths, though.

Stability constrains permissible merging patterns.  For example, if we have
3 consecutive runs of lengths

    A:10000  B:20000  C:10000

we dare not merge A with C first, because if A, B and C happen to contain
a common element, it would get out of order wrt its occurrence(s) in B.  The
merging must be done as (A+B)+C or A+(B+C) instead.

So merging is always done on two consecutive runs at a time, and in-place,
although this may require some temp memory (more on that later).

When a run is identified, its base address and length are pushed on a stack
in the MergeState struct.  merge_collapse() is then called to see whether it
should merge it with preceding run(s).  We would like to delay merging as
long as possible in order to exploit patterns that may come up later, but we
like even more to do merging as soon as possible to exploit that the run just
found is still high in the memory hierarchy.  We also can't delay merging
"too long" because it consumes memory to remember the runs that are still
unmerged, and the stack has a fixed size.

What turned out to be a good compromise maintains two invariants on the
stack entries, where A, B and C are the lengths of the three righmost not-yet
merged slices:

1.  A > B+C
2.  B > C

Note that, by induction, #2 implies the lengths of pending runs form a
decreasing sequence.  #1 implies that, reading the lengths right to left,
the pending-run lengths grow at least as fast as the Fibonacci numbers.
Therefore the stack can never grow larger than about log_base_phi(N) entries,
where phi = (1+sqrt(5))/2 ~= 1.618.  Thus a small # of stack slots suffice
for very large arrays.

If A <= B+C, the smaller of A and C is merged with B (ties favor C, for the
freshness-in-cache reason), and the new run replaces the A,B or B,C entries;
e.g., if the last 3 entries are

    A:30  B:20  C:10

then B is merged with C, leaving

    A:30  BC:30

on the stack.  Or if they were

    A:500  B:400:  C:1000

then A is merged with B, leaving

    AB:900  C:1000

on the stack.

In both examples, the stack configuration after the merge still violates
invariant #2, and merge_collapse() goes on to continue merging runs until
both invariants are satisfied.  As an extreme case, suppose we didn't do the
minrun gimmick, and natural runs were of lengths 128, 64, 32, 16, 8, 4, 2,
and 2.  Nothing would get merged until the final 2 was seen, and that would
trigger 7 perfectly balanced merges.

The thrust of these rules when they trigger merging is to balance the run
lengths as closely as possible, while keeping a low bound on the number of
runs we have to remember.  This is maximally effective for random data,
where all runs are likely to be of (artificially forced) length minrun, and
then we get a sequence of perfectly balanced merges (with, perhaps, some
oddballs at the end).

OTOH, one reason this sort is so good for partly ordered data has to do
with wildly unbalanced run lengths.


Merge Memory
------------
Merging adjacent runs of lengths A and B in-place, and in linear time, is
difficult.  Theoretical constructions are known that can do it, but they're
too difficult and slow for practical use.  But if we have temp memory equal
to min(A, B), it's easy.

If A is smaller (function merge_lo), copy A to a temp array, leave B alone,
and then we can do the obvious merge algorithm left to right, from the temp
area and B, starting the stores into where A used to live.  There's always a
free area in the original area comprising a number of elements equal to the
number not yet merged from the temp array (trivially true at the start;
proceed by induction).  The only tricky bit is that if a comparison raises an
exception, we have to remember to copy the remaining elements back in from
the temp area, lest the array end up with duplicate entries from B.  But
that's exactly the same thing we need to do if we reach the end of B first,
so the exit code is pleasantly common to both the normal and error cases.

If B is smaller (function merge_hi, which is merge_lo's "mirror image"),
much the same, except that we need to merge right to left, copying B into a
temp array and starting the stores at the right end of where B used to live.

A refinement:  When we're about to merge adjacent runs A and B, we first do
a form of binary search (more on that later) to see where B[0] should end up
in A.  Elements in A preceding that point are already in their final
positions, effectively shrinking the size of A.  Likewise we also search to
see where A[-1] should end up in B, and elements of B after that point can
also be ignored.  This cuts the amount of temp memory needed by the same
amount.

These preliminary searches may not pay off, and can be expected *not* to
repay their cost if the data is random.  But they can win huge in all of
time, copying, and memory savings when they do pay, so this is one of the
"per-merge overheads" mentioned above that we're happy to endure because
there is at most one very short run.  It's generally true in this algorithm
that we're willing to gamble a little to win a lot, even though the net
expectation is negative for random data.


Merge Algorithms
----------------
merge_lo() and merge_hi() are where the bulk of the time is spent.  merge_lo
deals with runs where A <= B, and merge_hi where A > B.  They don't know
whether the data is clustered or uniform, but a lovely thing about merging
is that many kinds of clustering "reveal themselves" by how many times in a
row the winning merge element comes from the same run.  We'll only discuss
merge_lo here; merge_hi is exactly analogous.

Merging begins in the usual, obvious way, comparing the first element of A
to the first of B, and moving B[0] to the merge area if it's less than A[0],
else moving A[0] to the merge area.  Call that the "one pair at a time"
mode.  The only twist here is keeping track of how many times in a row "the
winner" comes from the same run.

If that count reaches MIN_GALLOP, we switch to "galloping mode".  Here
we *search* B for where A[0] belongs, and move over all the B's before
that point in one chunk to the merge area, then move A[0] to the merge
area.  Then we search A for where B[0] belongs, and similarly move a
slice of A in one chunk.  Then back to searching B for where A[0] belongs,
etc.  We stay in galloping mode until both searches find slices to copy
less than MIN_GALLOP elements long, at which point we go back to one-pair-
at-a-time mode.

A refinement:  The MergeState struct contains the value of min_gallop that
controls when we enter galloping mode, initialized to MIN_GALLOP.
merge_lo() and merge_hi() adjust this higher when galloping isn't paying
off, and lower when it is.


Galloping
---------
Still without loss of generality, assume A is the shorter run.  In galloping
mode, we first look for A[0] in B.  We do this via "galloping", comparing
A[0] in turn to B[0], B[1], B[3], B[7], ..., B[2**j - 1], ..., until finding
the k such that B[2**(k-1) - 1] < A[0] <= B[2**k - 1].  This takes at most
roughly lg(B) comparisons, and, unlike a straight binary search, favors
finding the right spot early in B (more on that later).

After finding such a k, the region of uncertainty is reduced to 2**(k-1) - 1
consecutive elements, and a straight binary search requires exactly k-1
additional comparisons to nail it (see note REGION OF UNCERTAINTY).  Then we
copy all the B's up to that point in one chunk, and then copy A[0].  Note
that no matter where A[0] belongs in B, the combination of galloping + binary
search finds it in no more than about 2*lg(B) comparisons.

If we did a straight binary search, we could find it in no more than
ceiling(lg(B+1)) comparisons -- but straight binary search takes that many
comparisons no matter where A[0] belongs.  Straight binary search thus loses
to galloping unless the run is quite long, and we simply can't guess
whether it is in advance.

If data is random and runs have the same length, A[0] belongs at B[0] half
the time, at B[1] a quarter of the time, and so on:  a consecutive winning
sub-run in B of length k occurs with probability 1/2**(k+1).  So long
winning sub-runs are extremely unlikely in random data, and guessing that a
winning sub-run is going to be long is a dangerous game.

OTOH, if data is lopsided or lumpy or contains many duplicates, long
stretches of winning sub-runs are very likely, and cutting the number of
comparisons needed to find one from O(B) to O(log B) is a huge win.

Galloping compromises by getting out fast if there isn't a long winning
sub-run, yet finding such very efficiently when they exist.

I first learned about the galloping strategy in a related context; see:

    "Adaptive Set Intersections, Unions, and Differences" (2000)
    Erik D. Demaine, Alejandro López-Ortiz, J. Ian Munro

and its followup(s).  An earlier paper called the same strategy
"exponential search":

   "Optimistic Sorting and Information Theoretic Complexity"
   Peter McIlroy
   SODA (Fourth Annual ACM-SIAM Symposium on Discrete Algorithms), pp
   467-474, Austin, Texas, 25-27 January 1993.

and it probably dates back to an earlier paper by Bentley and Yao.  The
McIlroy paper in particular has good analysis of a mergesort that's
probably strongly related to this one in its galloping strategy.


Galloping with a Broken Leg
---------------------------
So why don't we always gallop?  Because it can lose, on two counts:

1. While we're willing to endure small per-merge overheads, per-comparison
   overheads are a different story.  Calling Yet Another Function per
   comparison is expensive, and gallop_left() and gallop_right() are
   too long-winded for sane inlining.

2. Galloping can-- alas --require more comparisons than linear one-at-time
   search, depending on the data.

#2 requires details.  If A[0] belongs before B[0], galloping requires 1
compare to determine that, same as linear search, except it costs more
to call the gallop function.  If A[0] belongs right before B[1], galloping
requires 2 compares, again same as linear search.  On the third compare,
galloping checks A[0] against B[3], and if it's <=, requires one more
compare to determine whether A[0] belongs at B[2] or B[3].  That's a total
of 4 compares, but if A[0] does belong at B[2], linear search would have
discovered that in only 3 compares, and that's a huge loss!  Really.  It's
an increase of 33% in the number of compares needed, and comparisons are
expensive in Python.

index in B where    # compares linear  # gallop  # binary  gallop
A[0] belongs        search needs       compares  compares  total
----------------    -----------------  --------  --------  ------
               0                    1         1         0       1

               1                    2         2         0       2

               2                    3         3         1       4
               3                    4         3         1       4

               4                    5         4         2       6
               5                    6         4         2       6
               6                    7         4         2       6
               7                    8         4         2       6

               8                    9         5         3       8
               9                   10         5         3       8
              10                   11         5         3       8
              11                   12         5         3       8
                                        ...

In general, if A[0] belongs at B[i], linear search requires i+1 comparisons
to determine that, and galloping a total of 2*floor(lg(i))+2 comparisons.
The advantage of galloping is unbounded as i grows, but it doesn't win at
all until i=6.  Before then, it loses twice (at i=2 and i=4), and ties
at the other values.  At and after i=6, galloping always wins.

We can't guess in advance when it's going to win, though, so we do one pair
at a time until the evidence seems strong that galloping may pay.  MIN_GALLOP
is 7, and that's pretty strong evidence.  However, if the data is random, it
simply will trigger galloping mode purely by luck every now and again, and
it's quite likely to hit one of the losing cases next.  On the other hand,
in cases like ~sort, galloping always pays, and MIN_GALLOP is larger than it
"should be" then.  So the MergeState struct keeps a min_gallop variable
that merge_lo and merge_hi adjust:  the longer we stay in galloping mode,
the smaller min_gallop gets, making it easier to transition back to
galloping mode (if we ever leave it in the current merge, and at the
start of the next merge).  But whenever the gallop loop doesn't pay,
min_gallop is increased by one, making it harder to transition back
to galloping mode (and again both within a merge and across merges).  For
random data, this all but eliminates the gallop penalty:  min_gallop grows
large enough that we almost never get into galloping mode.  And for cases
like ~sort, min_gallop can fall to as low as 1.  This seems to work well,
but in all it's a minor improvement over using a fixed MIN_GALLOP value.


Galloping Complication
----------------------
The description above was for merge_lo.  merge_hi has to merge "from the
other end", and really needs to gallop starting at the last element in a run
instead of the first.  Galloping from the first still works, but does more
comparisons than it should (this is significant -- I timed it both ways). For
this reason, the gallop_left() and gallop_right() (see note LEFT OR RIGHT)
functions have a "hint" argument, which is the index at which galloping
should begin.  So galloping can actually start at any index, and proceed at
offsets of 1, 3, 7, 15, ... or -1, -3, -7, -15, ... from the starting index.

In the code as I type it's always called with either 0 or n-1 (where n is
the # of elements in a run).  It's tempting to try to do something fancier,
melding galloping with some form of interpolation search; for example, if
we're merging a run of length 1 with a run of length 10000, index 5000 is
probably a better guess at the final result than either 0 or 9999.  But
it's unclear how to generalize that intuition usefully, and merging of
wildly unbalanced runs already enjoys excellent performance.

~sort is a good example of when balanced runs could benefit from a better
hint value:  to the extent possible, this would like to use a starting
offset equal to the previous value of acount/bcount.  Doing so saves about
10% of the compares in ~sort.  However, doing so is also a mixed bag,
hurting other cases.


Comparing Average # of Compares on Random Arrays
------------------------------------------------
[NOTE:  This was done when the new algorithm used about 0.1% more compares
 on random data than does its current incarnation.]

Here list.sort() is samplesort, and list.msort() this sort:

"""
import random
from time import clock as now

def fill(n):
    from random import random
    return [random() for i in xrange(n)]

def mycmp(x, y):
    global ncmp
    ncmp += 1
    return cmp(x, y)

def timeit(values, method):
    global ncmp
    X = values[:]
    bound = getattr(X, method)
    ncmp = 0
    t1 = now()
    bound(mycmp)
    t2 = now()
    return t2-t1, ncmp

format = "%5s  %9.2f  %11d"
f2     = "%5s  %9.2f  %11.2f"

def drive():
    count = sst = sscmp = mst = mscmp = nelts = 0
    while True:
        n = random.randrange(100000)
        nelts += n
        x = fill(n)

        t, c = timeit(x, 'sort')
        sst += t
        sscmp += c

        t, c = timeit(x, 'msort')
        mst += t
        mscmp += c

        count += 1
        if count % 10:
            continue

        print "count", count, "nelts", nelts
        print format % ("sort",  sst, sscmp)
        print format % ("msort", mst, mscmp)
        print f2     % ("", (sst-mst)*1e2/mst, (sscmp-mscmp)*1e2/mscmp)

drive()
"""

I ran this on Windows and kept using the computer lightly while it was
running.  time.clock() is wall-clock time on Windows, with better than
microsecond resolution.  samplesort started with a 1.52% #-of-comparisons
disadvantage, fell quickly to 1.48%, and then fluctuated within that small
range.  Here's the last chunk of output before I killed the job:

count 2630 nelts 130906543
 sort    6110.80   1937887573
msort    6002.78   1909389381
            1.80         1.49

We've done nearly 2 billion comparisons apiece at Python speed there, and
that's enough <wink>.

For random arrays of size 2 (yes, there are only 2 interesting ones),
samplesort has a 50%(!) comparison disadvantage.  This is a consequence of
samplesort special-casing at most one ascending run at the start, then
falling back to the general case if it doesn't find an ascending run
immediately.  The consequence is that it ends up using two compares to sort
[2, 1].  Gratifyingly, timsort doesn't do any special-casing, so had to be
taught how to deal with mixtures of ascending and descending runs
efficiently in all cases.


NOTES
-----

BINSORT
A "binary insertion sort" is just like a textbook insertion sort, but instead
of locating the correct position of the next item via linear (one at a time)
search, an equivalent to Python's bisect.bisect_right is used to find the
correct position in logarithmic time.  Most texts don't mention this
variation, and those that do usually say it's not worth the bother:  insertion
sort remains quadratic (expected and worst cases) either way.  Speeding the
search doesn't reduce the quadratic data movement costs.

But in CPython's case, comparisons are extraordinarily expensive compared to
moving data, and the details matter.  Moving objects is just copying
pointers.  Comparisons can be arbitrarily expensive (can invoke arbitrary
user-supplied Python code), but even in simple cases (like 3 < 4) _all_
decisions are made at runtime:  what's the type of the left comparand?  the
type of the right?  do they need to be coerced to a common type?  where's the
code to compare these types?  And so on.  Even the simplest Python comparison
triggers a large pile of C-level pointer dereferences, conditionals, and
function calls.

So cutting the number of compares is almost always measurably helpful in
CPython, and the savings swamp the quadratic-time data movement costs for
reasonable minrun values.


LEFT OR RIGHT
gallop_left() and gallop_right() are akin to the Python bisect module's
bisect_left() and bisect_right():  they're the same unless the slice they're
searching contains a (at least one) value equal to the value being searched
for.  In that case, gallop_left() returns the position immediately before the
leftmost equal value, and gallop_right() the position immediately after the
rightmost equal value.  The distinction is needed to preserve stability.  In
general, when merging adjacent runs A and B, gallop_left is used to search
thru B for where an element from A belongs, and gallop_right to search thru A
for where an element from B belongs.


REGION OF UNCERTAINTY
Two kinds of confusion seem to be common about the claim that after finding
a k such that

    B[2**(k-1) - 1] < A[0] <= B[2**k - 1]

then a binary search requires exactly k-1 tries to find A[0]'s proper
location. For concreteness, say k=3, so B[3] < A[0] <= B[7].

The first confusion takes the form "OK, then the region of uncertainty is at
indices 3, 4, 5, 6 and 7:  that's 5 elements, not the claimed 2**(k-1) - 1 =
3"; or the region is viewed as a Python slice and the objection is "but that's
the slice B[3:7], so has 7-3 = 4 elements".  Resolution:  we've already
compared A[0] against B[3] and against B[7], so A[0]'s correct location is
already known wrt _both_ endpoints.  What remains is to find A[0]'s correct
location wrt B[4], B[5] and B[6], which spans 3 elements.  Or in general, the
slice (leaving off both endpoints) (2**(k-1)-1)+1 through (2**k-1)-1
inclusive = 2**(k-1) through (2**k-1)-1 inclusive, which has
    (2**k-1)-1 - 2**(k-1) + 1 =
    2**k-1 - 2**(k-1) =
    2*2**k-1 - 2**(k-1) =
    (2-1)*2**(k-1) - 1 =
    2**(k-1) - 1
elements.

The second confusion:  "k-1 = 2 binary searches can find the correct location
among 2**(k-1) = 4 elements, but you're only applying it to 3 elements:  we
could make this more efficient by arranging for the region of uncertainty to
span 2**(k-1) elements."  Resolution:  that confuses "elements" with
"locations".  In a slice with N elements, there are N+1 _locations_.  In the
example, with the region of uncertainty B[4], B[5], B[6], there are 4
locations:  before B[4], between B[4] and B[5], between B[5] and B[6], and
after B[6].  In general, across 2**(k-1)-1 elements, there are 2**(k-1)
locations.  That's why k-1 binary searches are necessary and sufficient.

```

