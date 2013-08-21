.. module:: pulsar

.. _tutorials-coroutine:

=========================
Asynchronous Components
=========================

There are three, closely related, ways to create asynchronous components in
pulsar:

* Directly create an instance of :class:`Deferred`::
  
      import pulsar
      
      o = pulsar.Deferred()
      
* A :ref:`coroutine <coroutine>`, a generator which consumes values.
  For example::
  
      def my_async_generator(...):
          yield something_but_dont_care_what_it_returns()
          ...
          bla = yield something_and_care_what_it_returns()
          yield do_something(bla)

  a coroutine is obtained by calling the generator function::
  
      o = my_async_generator()
  
  Note that ``o`` is coroutine which has not yet started.
  
* A :class:`Task`, is a component which has been added to pulsar asynchronous
  engine. It is created via the :class:`async` decorator::
  
      from pulsar import async
      
      @async()
      def my_async_generator(...):
          yield something_but_dont_care_what_it_returns()
          ...
          bla = yield something_and_care_what_it_returns()
          yield do_something(bla)
  
      task = my_async_generator()
      
  A :class:`Task` is a subclass of :class:`Deferred` and therefore it has
  the same API, for example, you can add callbacks to a task::
  
      task.add_callback(...)
 

.. _deferred:
  
Deferred
===================
A :class:`Deferred` is a callback which will be put off until later. its
implementation is similar to the `twisted deferred`_ class with few
important differences. A deferred is the product of an asynchronous operation.

.. _deferred-event-loop:

The event loop
~~~~~~~~~~~~~~~~~~~~~~~

A :class:`Deferred` has accessed to the event loop via the
:attr:`Deferred.event_loop` attribute. The event loop can be set during
initialisation::

    d = Deferred(event_loop=loop)

If not set, it is obtained using the ``get_event_loop`` function.

A vanilla :class:`Deferred` needs the event loop only when invoking
the :meth:`Deferred.set_timeout` method. On the other hand, A :class:`Task`
(a :class:`Deferred` which consumes a :ref:`coroutine <coroutine>`) requires
it during initialisation.

.. _deferred-cancel:

Cancelling a deferred
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Deferreds can be cancelled, for example when an operation is taking too long to
finish. To cancel a deferred one invokes the :meth:`Deferred.cancel`
method. Calling ``cancel`` on an already called or cancelled deferred
has no effect, therefore the :meth:`Deferred.cancel` will always
succeed.

When a :class:`Task` is cancelled, the deferred on which the task is blocked is
cancelled too. For example::

    >>> from pulsar import Deferred, maybe_async  
    >>> d = Deferred()
    >>> def gen():
    ...     yield d 
    >>> task = maybe_async(gen())
    >>> task.cancel()
    >>> task.cancelled()
    True
    >>> d.cancelled()
    True
    >>> d.done()
    True

Timeouts
~~~~~~~~~~~~~~
A useful application of :ref:`deferred cancellation <deferred-cancel>`,
is setting a ``timeout`` to an asynchronous operation. To set a timeout,
one used the :class:`Deferred.set_timeout` method::

    >>> d = Deferred()
    >>> d.set_timeout(5)
    
To avoid cancelling the underlying operation one could use this trick::

    d2 = d1.then().set_timeout(5)
    
or a double layer timeout::

    d2 = d1.set_timeout(10).then().set_timeout(5)
    
.. _coroutine:
  
Coroutines
===================
As mentioned above a coroutine is a generator which consumes values. A pulsar
coroutine can consume synchronous values as well as :class:`Deferred` and
other :ref:`coroutines <coroutine>`.
Let's consider the following code::

    d = Deferred()
    
    def do_something(...):
          yield something_but_dont_care_what_it_returns()
          ...
          bla = yield something_and_care_what_it_returns()
          yield do_something(bla)
          
    def my_async_generator():
          result = yield d
          yield do_something(result)
          
Then we create a coroutine by calling the ``my_async_generator`` generator
function::

    o = my_async_generator()
    
``o`` is has not yet started. To use it, it must be added to pulsar
asynchronous engine via the :func:`maybe_async` function::

    task = maybe_async(o, get_result=False)

task is a :class:`Task` instance.

Task
===================
A :class:`Task` is a specialised :class:`Deferred` which consumes
:ref:`coroutines <coroutine>`.
A coroutine is transformed into a :class:`Task`
via the :func:`maybe_async` function or the :class:`async` decorator.

A task consumes a coroutine until the coroutine yield an asynchronous component
not yet done. When this appends, the task pauses and returns the control of execution.
Before it returns, it adds a ``callback`` (and ``errback``) to the :class:`Deferred`
on which the coroutine is blocked to resume the coroutine once the deferred
is called. 
A task in this state is said to be **suspended**.
    

Collections
============================
When dealing with several asynchronous components in a collection such as
a list, tuple, set or even a dictionary (values only, keys must be synchronous
python types), one can use the :func:`multi_async` function to create
an asynchronous component which will be ready once all the components
are ready.


.. _twisted deferred: http://twistedmatrix.com/documents/current/core/howto/defer.html