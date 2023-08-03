# Benefiting From Threading in Spite of the GIL in Python

As many people use Python on a daily basis, I thought I'd share something that interested me for quite some time, and
very recently I finally took initiative to get to the bottom of.

!!! note "Disclaimer"

    This isn't about best practices or common use-cases, it's a bit deeper dive, to those who take interest in Python and
    its internals (_very deep_ internals) and care about how and why.

    If it's not very interesting to you, it's totally cool, you can stop here, `exit()`.

## The Scope

The most wide-spread implementation of Python is CPython (written in C, duh), it's open-source, and here's the repo:
[python/cpython](https://github.com/python/cpython). That's where my research took place.

Note that currently Python 3.13 is underway (this document is written over CPython with
the [commit SHA of 557b05c](https://github.com/python/cpython/tree/557b05c7a5334de5da3dc94c108c0121f10b9191)), and for
persistence the links are in relation to that specific commit, implementation may vary (greatly) with time, so the
specifics and possible change between versions are omitted, as this document is powered by curiosity alone.

## Threads/Asyncio

Python (currently) works with a single process that can share resources with multiple threads, it manages to do so with
the [GIL](https://docs.python.org/3/glossary.html#term-global-interpreter-lock) (Global Interpreter Lock), with each
thread awaiting its turn ([PEP 703](https://peps.python.org/pep-0703/) anyone?).

Many of us have had some experience with async functions, especially in writing APIs or in the context of network
interaction, and expect the speedup of using it in specific areas, like communication with remote servers, writing logs,
etc.

While threads and asyncio aren't the same thing, both have ways of improving performance in certain areas, by allowing
other tasks to execute at the same time for specific operations.

This may raise a few questions, like _"Why do we get a speedup there?"_ or _"How does Python know where to release the
GIL, and let other tasks resume?"_.

## Under The Hood

Well, there isn't such thing as magic (not if you go deep enough, which in some cases you simply shouldn't), so here's
the process I went through to look for an answer.

1. I searched for something I know should have a blocking system call somewhere, like reading from a file, so naturally
   I went to `Lib/pathlib.py`, and
   specifically [`pathlib.Path.read_bytes()`](https://github.com/python/cpython/tree/557b05c7a5334de5da3dc94c108c0121f10b9191/Lib/pathlib.py#L971).
   Which calls `pathlib.Path.open()`, which in turn calls `io.open()`, but looking at
   [`Lib/io.py`](https://github.com/python/cpython/tree/557b05c7a5334de5da3dc94c108c0121f10b9191/Lib/io.py) I couldn't
   see anything useful, so at this point, from my experience with CPython, I know I needed to look for the C code which
   actually performs the logic.

2. Conveniently enough, I found `Modules/_io/fileio.c` and the function inside
   it [`_io_FileIO_read_impl`](https://github.com/python/cpython/tree/557b05c7a5334de5da3dc94c108c0121f10b9191/Modules/_io/fileio.c#L798),
   which in turn
   calls [`_Py_read`](https://github.com/python/cpython/tree/557b05c7a5334de5da3dc94c108c0121f10b9191/Python/fileutils.c#L1847),
   and inside this function I stumbled upon something interesting...

3. 

    ```c title="Python/fileutils.c" linenums="1846" hl_lines="21 25 34 39"
    Py_ssize_t
    _Py_read(int fd, void *buf, size_t count)
    {
        Py_ssize_t n;
        int err;
        int async_err = 0;
    
        assert(PyGILState_Check());
    
        /* _Py_read() must not be called with an exception set, otherwise the
         * caller may think that read() was interrupted by a signal and the signal
         * handler raised an exception. */
        assert(!PyErr_Occurred());
    
        if (count > _PY_READ_MAX) {
            count = _PY_READ_MAX;
        }
    
        _Py_BEGIN_SUPPRESS_IPH
        do {
            Py_BEGIN_ALLOW_THREADS
            errno = 0;
    #ifdef MS_WINDOWS
            _doserrno = 0;
            n = read(fd, buf, (int)count);
            // read() on a non-blocking empty pipe fails with EINVAL, which is
            // mapped from the Windows error code ERROR_NO_DATA.
            if (n < 0 && errno == EINVAL) {
                if (_doserrno == ERROR_NO_DATA) {
                    errno = EAGAIN;
                }
            }
    #else
            n = read(fd, buf, count);
    #endif
            /* save/restore errno because PyErr_CheckSignals()
             * and PyErr_SetFromErrno() can modify it */
            err = errno;
            Py_END_ALLOW_THREADS
        } while (n < 0 && err == EINTR &&
                !(async_err = PyErr_CheckSignals()));
        _Py_END_SUPPRESS_IPH
    
        if (async_err) {
            /* read() was interrupted by a signal (failed with EINTR)
             * and the Python signal handler raised an exception */
            errno = err;
            assert(errno == EINTR && PyErr_Occurred());
            return -1;
        }
        if (n < 0) {
            PyErr_SetFromErrno(PyExc_OSError);
            errno = err;
            return -1;
        }
    
        return n;
    }
    ```

4. In lines 1866 and 1884 (highlighted) of the snippet above we see these two suspicious macros:
   `Py_BEGIN_ALLOW_THREADS` and `Py_END_ALLOW_THREADS` respectively, so I looked for where they're defined, which was
   [`Include/ceval.h`](https://github.com/python/cpython/blob/a1c737b73d3658be0e1d072a340d42e3d96373c6/Include/ceval.h#L65),
   and above these functions the documentation says:

    ```c title="Include/ceval.h" linenums="65"
    /* Interface for threads.
    
       A module that plans to do a blocking system call (or something else
       that lasts a long time and doesn't touch Python data) can allow other
       threads to run as follows:
    
       ...preparations here...
       Py_BEGIN_ALLOW_THREADS
       ...blocking system call here...
       Py_END_ALLOW_THREADS
       ...interpret result here...
    
    ...
    ```

5. So it seemed I found what I was looking for, these two macros define the start and end of a blocking system call, and
   code written between these can benefit our async Python code.
   And indeed, between these macros in lines 1870 and 1879 (also highlighted) of the snippet above lies the
   syscall [read](https://man7.org/linux/man-pages/man2/read.2.html).

6. I admit I did not really go deeper than this, below these macros are the C functions implementation and usage of
   internal CPython structures, so this is as far as this specific journey went.

## Verifying The Theory

I wanted to make sure I was right, and wanted to see another example of it in a different context where it makes sense,
and so I searched for these macros in CPython, and found them being used in `Modules/socketmodule.c` inside the function
[`sock_call_ex`](https://github.com/python/cpython/tree/557b05c7a5334de5da3dc94c108c0121f10b9191/Modules/socketmodule.c#L972)
(_call ex, hehe..._), which after some looking at the code seemed to be called with many socket related syscalls:
[`accept`](https://man7.org/linux/man-pages/man2/accept.2.html),
[`connect`](https://man7.org/linux/man-pages/man2/connect.2.html),
[`send`](https://man7.org/linux/man-pages/man2/send.2.html),
[`recv`](https://man7.org/linux/man-pages/man2/recv.2.html), and more.

This was indeed reassuring of the theory above, and I concluded my research happily.

## Myth-Busting :material-ghost-off:

Calling C code (or any other language) from Python does not guarantee concurrency, as written in Python's code about the
GIL: _"The mechanism used by the CPython interpreter to assure that only one thread executes Python bytecode at a
time"_.

The purpose is to protect Python data from being modified by several threads simultaneously, meaning that the release of
the GIL needs to be explicit, and of course, with much caution.

## Bonus

In [`Include/ceval.h`](https://github.com/python/cpython/blob/a1c737b73d3658be0e1d072a340d42e3d96373c6/Include/ceval.h#L107)
the macros were defined as:

```c title="Include/ceval.h" linenums="107"
PyAPI_FUNC(PyThreadState *) PyEval_SaveThread(void);
PyAPI_FUNC(void) PyEval_RestoreThread(PyThreadState *);

PyAPI_FUNC(void) PyEval_AcquireThread(PyThreadState *tstate);
PyAPI_FUNC(void) PyEval_ReleaseThread(PyThreadState *tstate);

#define Py_BEGIN_ALLOW_THREADS { \
                 PyThreadState *_save; \
                 _save = PyEval_SaveThread();
#define Py_BLOCK_THREADS      PyEval_RestoreThread(_save);
#define Py_UNBLOCK_THREADS     _save = PyEval_SaveThread();
#define Py_END_ALLOW_THREADS   PyEval_RestoreThread(_save); \
            }
```

I took a peek at `PyEval_SaveThread`, which I found in
[`Python/ceval_gil.c`](https://github.com/python/cpython/tree/557b05c7a5334de5da3dc94c108c0121f10b9191/Python/ceval_gil.c#L729),
and there I saw:

```c title="Python/ceval_gil.c" linenums="728"
PyThreadState *
PyEval_SaveThread(void)
{
    PyThreadState *tstate = _PyThreadState_SwapNoGIL(NULL);
    _Py_EnsureTstateNotNULL(tstate);

    struct _ceval_state *ceval = &tstate->interp->ceval;
    assert(gil_created(ceval->gil));
    drop_gil(ceval, tstate);
    return tstate;
}
```

This gave me a glance at the existence of functions like `drop_gil`, `gil_created`, `destroy_gil`, and `take_gil`.
All there in that file.

So if anyone wants to go even deeper, here's where you can continue from.

## Conclusion

I like looking inside Python and see how it works. For me, it was a very interesting research, and I hope it was
interesting to you as well.

!!! quote

    _"There's no such thing as magic!"_ - Vernon Dursley, Harry Potter.
