.. -*- coding: utf-8 -*-


A tale of event loops
=====================

I was recently stuck by the *amazing* insight from `Nathaniel J. Smith`_, a
fellow at Berkely, about asyncio_.  His post `Some thoughts on asynchronous API
design in a post-async/await world`_ helped me put some words on the (slightly)
mixed feelings I have about asyncio after using it a lot recently.

.. _`Nathaniel J. Smith`: https://vorpus.org/
.. _asyncio: https://docs.python.org/3/library/asyncio.html
.. _`Some thoughts on asynchronous API design in a post-async/await world`: https://vorpus.org/blog/some-thoughts-on-asynchronous-api-design-in-a-post-asyncawait-world/

While this is more or less the main focus in his post, he argues that curio_
has a much simpler implementation than asyncio_ because it assumes upfront that
you have a version of Python that implements `PEP 492`_ (``async`` and
``await`` keywords).

.. _curio: https://github.com/dabeaz/curio
.. _`PEP 492`: https://www.python.org/dev/peps/pep-0492/

I was intrigued and took a look at curio's source and... due to it gaining a
lot of features (hopefully briging it closer to a production-ready library), I
think it has enough features now that the main essence is diluted.  This is
definitely *not* a bad thing, but if, like me, you'd like to understand what
Nathaniel was talking about, and learn a few neat Python tricks, reading
curio's source code is hardly a good starting point.

`Brett Cannon`_, a Python core developer gives some very good insight on how to
use coroutine objects (e.g. if you were implementing an event loop) in his post
`How the heck does async/await work inPython 3.5?`_.  His focus is on the
implementation details and, at least for me, there was still a small piece
missing to grok curio.

.. _`Brett Cannon`: http://www.snarky.ca/
.. _`How the heck does async/await work inPython 3.5?`: http://www.snarky.ca/how-the-heck-does-async-await-work-in-python-3-5

**TL; DR**: This is my attempt at trying to capture the core fundamentals on
which curio is based.  Hopefully, it will be a good stepping stone to get
acquainted with curio's source code.

Let's get started :-)


The real nature of coroutines
-----------------------------

First, let's take a look at what happens when you call a coroutine.

**Hint**: you get a coroutine *object*, which has a ``.send()`` and a
 ``.throw()`` method, just like generator objects do.

.. code:: python

   >>> async def hello(name):
   ...     print('Hello, %s!' % (name,))
   >>>
   >>> coro = hello('world')
   >>> type(coro)
   <class 'coroutine'>
   >>> type(coro.send)
   <class 'builtin_function_or_method'>
   >>> type(coro.throw)
   <class 'builtin_function_or_method'>
   >>>
   ...

Of course you usually don't use these methods because they are hidden behind
``await coro`` and ``asyncio.get_event_loop().run_until_complete()``, but since
we're trying to look into how *that* works... :-)

Notice that the code above never printed our ``"Hello, world!"`` message.
That's because we never actually executed statements inside the coroutine
function -- we simply created the coroutine object.  In fact, if you actually
run this code in an interpreter, you will get a warning stating that our
``hello`` coroutine never completed.

If you want to execute the coroutine function, you need to schedule it somehow.
In order to do that, we're going to call the ``.send()`` method.

.. code:: python

   >>> async def hello(name):
   ...     print('Hello, %s!' % (name,))
   >>>
   >>> task = hello('world')
   >>> try:
   ...     task.send(None)
   ... except StopIteration:
   ...     pass
   Hello, world!

Just like for a generator object, a couroutine object's ``.send()`` method
raises ``StopIteration`` when the coroutine ends.

We'll see later that we can use ``.send()`` to pass information to the
coroutine when resuming it with a 2nd, 3rd or Nth call to ``.send()``.

If, instead of returning normally, the coroutine fails with an exception, then
that exception will be propagated back through ``.send()``.

   >>> class Hello(Exception):
   ...    def __init__(self, name):
   ...        self._name = name
   ...    def __str__(self):
   ...        return 'Hello, %s!' % (self._name,)
   >>>
   >>> async def hello(name):
   ...     raise Hello(name)
   >>>
   >>> task = hello('world')
   >>> try:
   ...     task.send(None)
   ... except Exception as error:
   ...     # NEW: exception will be propagated here!
   ...     print(error)
   Hello, world!

I also mentioned a ``.throw()`` method.  Like ``.send()``, it resumes the
coroutine, but instead of passing it a value, it raises an exception inside the
coroutine at the point where it was suspended.

   >>> class Hello(Exception):
   ...    def __init__(self, name):
   ...        self._name = name
   ...    def __str__(self):
   ...        return 'Hello, %s!' % (self._name,)
   >>>
   >>> async def hello():
   ...     pass
   >>>
   >>> task = hello()
   >>> try:
   ...     # NEW: inject exception.
   ...     task.throw(Hello('world'))
   ... except Exception as error:
   ...     print(error)
   Hello, world!

At this point, you should be comfortable with the fact that coroutine objects
are very, very similar to generator objects which exist since Python 2.2 (`PEP
255`_) and have had ``.send()`` and ``.throw()`` methods since Python 2.5 (`PEP
342`_).

.. _`PEP 255`: https://www.python.org/dev/peps/pep-0255/
.. _`PEP 342`: https://www.python.org/dev/peps/pep-0342/


A dialog with the event loop
----------------------------

If you look around (or try it), you will notice that coroutine functions, in
contast with generator functions, cannot use the ``yield`` expression.  This
raises (*not* begs_) the question: how exactly can coroutine functions yield
control back to the code that called ``.send()``?

.. _begs: http://grammarist.com/usage/begging-the-question-usage/

The answer implies using ``await`` on an *awaitable* object.  For an object to
be awaitable, it must implement the special ``__await__()`` method that returns
an iterable.  In practice, this is a little awkward, so there is a
`@types.coroutine` decorator in the standard library that allows you to create
awaitable objects in a style reminescent of `@contextlib.contextmanager`_.

.. _`@types.coroutine`: https://docs.python.org/3.5/library/types.html#types.coroutine_
.. _`@contextlib.contextmanager`: https://docs.python.org/3.5/library/contextlib.html#contextlib.contextmanager

.. code:: python

   >>> from types import coroutine
   >>>
   >>> # NEW: this is an awaitable object!
   >>> @coroutine
   ... def nice():
   ...     yield
   >>>
   >>> async def hello(name):
   ...     # NEW: this makes ``.send()`` return!
   ...     await nice()
   ...     print('Hello, %s!' % (name,))
   >>>
   >>> task = hello('world')
   >>> # NEW: call send twice!
   >>> task.send(None)
   >>> try:
   ...     task.send(None)
   ... except StopIteration:
   ...     pass
   Hello, world!

 Of course, our ``nice()`` object is pretty useless.  Don't fret, we'll be
 doing some more useful things shortly.


Looping
-------

Our previous example calls ``.send()`` exactly twice because it nows that
``hello()`` will only yield control once.  When we don't know (the common
case), we need to put this in a loop.

.. code:: python

   >>> from types import coroutine
   >>>
   >>> @coroutine
   ... def nice():
   ...     yield
   >>>
   >>> async def hello(name):
   ...     # NEW: yield many times!
   ...     for i in range(5):
   ...         await nice()
   ...     print('Hello, %s!' % (name,))
   >>>
   >>> task = hello('world')
   >>> try:
   ...     # NEW: loop!
   ...     while True:
   ...         task.send(None)
   ... except StopIteration:
   ...     pass
   Hello, world!

We're starting to get something that feels like the simplest possible
implementation of ``asyncio.get_event_loop().run_until_complete()``, so let's
make it more syntactically similar.

.. code:: python

   >>> from types import coroutine
   >>>
   >>> @coroutine
   ... def nice():
   ...     yield
   >>>
   >>> async def hello(name):
   ...     for i in range(5):
   ...         await nice()
   ...     print('Hello, %s!' % (name,))
   >>>
   >>> # NEW: now a reusable function!
   >>> def run_until_complete(task):
   ...     try:
   ...         while True:
   ...             task.send(None)
   ...     except StopIteration:
   ...         pass
   >>>
   >>> # NEW: call it as a function!
   >>> run_until_complete(hello('world'))
   Hello, world!


Spawning child tasks
--------------------

Now that we've got an event loop that can complete a single task, we'll
probably want to start doing useful things.  There are many different things we
expect to do, but since this is about concurrency, let's start by allowing
creation of child tasks.

The main thing we need to do here is to introduce a new ``spawn()`` primitive
that schedules the new child task.  Once the task is scheduled, we want to
return control to the parent task so that it can continue on its way.

**Note**: this example is deliberaly incomplete.  We'll see how to join tasks
later.

.. code:: python

   >>> from inspect import iscoroutine
   >>> from types import coroutine
   >>>
   >>> # NEW: awaitable object that sends a request to launch a child task!
   >>> @coroutine
   ... def spawn(task):
   ...     yield task
   >>>
   >>> async def hello(name):
   ...     await nice()
   ...     print('Hello, %s!' % (name,))
   >>>
   >>> # NEW: parent task!
   >>> async def main():
   ...      # NEW: create a child task!
   ...     await spawn(hello('world'))
   >>>
   >>> def run_until_complete(task):
   ...     # NEW: schedule the "root" task.
   ...     tasks = [(task, None)]
   ...     while tasks:
   ...         # NEW: round-robin between a set tasks (we may now have more
   ...         #      than one and we'd like to be as "fair" as possible).
   ...         queue, tasks = tasks, []
   ...         for task, data in queue:
   ...             # NEW: resume the task *once*.
   ...             try:
   ...                 data = task.send(data)
   ...             except StopIteration:
   ...                 pass
   ...             except Exception as error:
   ...                 # NEW: prevent crashed task from ending the loop.
   ...                 print(error)
   ...             else:
   ...                 # NEW: schedule the child task.
   ...                 if iscoroutine(data):
   ...                     tasks.append((data, None))
   ...                 # NEW: reschedule the parent task.
   ...                 tasks.append((task, None))
   >>>
   >>> run_until_complete(main())
   Hello, world!

**Whoa!** This is way more complicated than our previous version of
``run_until_complete()``.  Where did all of *that* come from?

Well... now that we can run multiple tasks, we need to worry about things like:

#. waiting for all the child tasks to complete (recursively), despite errors in
   any of the tasks
#. alternating between tasks to let all tasks *concurrently* progress toward
   completion

Notice that we now have a nested loop:

* the outer loop takes care of checking if we're done
* the inner loop takes care of one scheduler "tick"

There are other ways to do this and it may not be immediately obvious why we're
doing it this way, but it's important because of two critical pieces of the
event loop that are still missing: timers and I/O.  When we add support these
later on, we're going to need to schedule internal checks in a "fair" manner
too.  The outer loop gives us a convenient spot for checking timers and polling
the status of I/O operations:

#. check timers to resume sleeping tasks for which the delay has elapsed;
#. check I/O operations and schedule tasks for which the pending I/O operations
   have completed;
#. perform one scheduler "tick" to resume all tasks we just scheduled.

In short, that's the gist of the coroutine-based scheduler loop.


**However**, before we get into the more complicated timers & I/O... remember
earlier when I mentionned that the example was deliberately incomplete?  We
know how to spawn new child tasks, but we don't yet know how to wait for them
to complete.  This a good opportunity to see how we extend the vocabulary of
event loop "requests" sent by our awaitable objects.

.. code:: python

   >>> from collections import defaultdict
   >>> from types import coroutine
   >>>
   >>> @coroutine
   ... def nice():
   ...     yield
   >>>
   >>> @coroutine
   ... def spawn(task):
   ...     # NEW: recover the child task handle to pass it back to the parent.
   ...     child = yield ('spawn', task)
   ...     return child
   >>>
   >>> # NEW: awaitable object that sends a request to be notified when a
   >>> #      concurrent task completes.
   >>> @coroutine
   ... def join(task):
   ...     yield ('join', task)
   >>>
   >>> async def hello(name):
   ...     await nice()
   ...     print('Hello, %s!' % (name,))
   >>>
   >>> async def main():
   ...     # NEW: recover the child task handle.
   ...     child = await spawn(hello('world'))
   ...     # NEW: wait for the child task to complete.
   ...     await join(child)
   ...     print('(after join)')
   >>>
   >>> def run_until_complete(task):
   ...     tasks = [(task, None)]
   ...     # NEW: keep track of tasks to resume when a task completes.
   ...     watch = defaultdict(list)
   ...     while tasks:
   ...         queue, tasks = tasks, []
   ...         for task, data in queue:
   ...             try:
   ...                 data = task.send(data)
   ...             except StopIteration:
   ...                 # NEW: wait up tasks waiting on this one.
   ...                 tasks.extend((t, None) for t in watch.pop(task, []))
   ...             else:
   ...                 # NEW: dispatch request sent by awaitable object since
   ...                 #      we now have 3 diffent types of requests.
   ...                 if data and data[0] == 'spawn':
   ...                     tasks.append((data[1], None))
   ...                     tasks.append((task, data[1]))
   ...                 elif data and data[0] == 'join':
   ...                     watch[data[1]].append(task)
   ...                 else:
   ...                     tasks.append((task, None))
   >>>
   >>> run_until_complete(main())
   Hello, world!
   (after join)

 For practical reasons, we'll probably want to have some kind ``Task`` wrapper
 for coroutine objects.  This comes handy to expose an API for cancellation and
 to handle some race conditions such as the child task ending before the parent
 task attempts to ``join()`` it (can you spot the bug?)

 Passing the child task's return value back as the result of ``await join()``
 and propagating the exception that crashed the child are left as exercices to
 the reader.


Sleeping & timers
-----------------

TODO


Handling I/O
------------

TODO


Cancelling tasks
----------------

TODO


License
-------

This document is copyright Andre Caron <andre.l.caron@gmail.com> and is made
available to you under a Creative Commons CC-BY-SA_ license.

.. _CC-BY-SA: https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode
