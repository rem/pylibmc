========================
 Pooling with `pylibmc` 
========================

:Author: Ludvig Ericson <ludvig circled-a lericson dot se>
:See also: `Pooling with pylibmc`__ (this document)
:See also: `Pooling with pylibmc pt. 2`__ (follow-up)

__ http://lericson.blogg.se/code/2009/september/draft-sept-20-2009.html
__ http://lericson.blogg.se/code/2009/september/pooling-with-pylibmc-pt-2.html

.. This is really a blog post, I do write them in ReST occasionally. Provided
   here for the sake of convenience.

.. Protip, you can make an HTML file out of this by executing:
     rst2html.py pooling.rst >pooling.html
   and then just
     open pooling.html

I was discussing how to implement pooling for `pylibmc` when I realized what
`libmemcachedutil`'s pooling is - or rather, what it isn't.

It's not a magical solution for concurrently doing anything at all, it's not
anything like that -- it just helps you with thread-safety.

In Python, however, we've got the global interpreter lock, the GIL. This lock
must always be held by the thread that is dealing with anything Python. The
Python interpreter itself isn't thread-safe, or rather, it is with the GIL.

This means that whenever Python code is running, you'll be sure to have
exclusive access to all of Python's memory (unless something is misbehaving.)
In turn, this means that the usecase for using `libmemcachedutil` in a Python
library is rather slim.

Let's have a look at some code for doing the equivalent in pure Python. This is
a Werkzeug-based WSGI application which would be run in multiple threads,
concurrently:

.. code-block:: python

    # Configuration
    n_threads = 12
    mc_addrs = "10.0.1.1", "10.0.1.2", "10.0.1.3"
    mc_pool_size = n_threads

    # Application
    import pylibmc
    from contextlib import contextmanager
    from pprint import pformat
    from werkzeug.wrappers import Request, Response
    from werkzeug.exceptions import NotFound

    class ClientPool(list):
        @contextmanager
        def reserve(self):
            mc = self.pop()
            try:
                yield mc
            finally:
                self.append(mc)

    mc = pylibmc.Client(mc_addrs)
    mc_pool = ClientPool(mc.clone() for i in xrange(mc_pool_size))

    @Request.application
    def my_app(request):
        with mc_pool.reserve() as mc:
            key = request.path[1:].encode("ascii")
            val = mc.get(key)
            if not val:
                return NotFound(key)
            return Response(pformat(val))

    if __name__ == "__main__":
        from werkzeug.serving import run_simple
        run_simple("0.0.0.0", 5050, my_app)

It's fully-functional example of how one could implement pooling with
`pylibmc`, and very much so in the same way that people do with
`libmemcachedutil`. To start it, just copy the code, run ``python <<PY``, paste
the code and type ``PY`` on a line of its own.

The above type of pool is actually already implemented in `pylibmc` as
``pylibmc.ClientPool`` -- it has a slightly different API for creation:

.. code-block:: python

    # Configuration
    n_threads = 12
    mc_addrs = "10.0.1.1", "10.0.1.2", "10.0.1.3"
    mc_pool_size = n_threads

    # Application
    import pylibmc
    from contextlib import contextmanager
    from pprint import pformat
    from werkzeug.wrappers import Request, Response
    from werkzeug.exceptions import NotFound

    mc = pylibmc.Client(mc_addrs)
    mc_pool = pylibmc.ClientPool()
    mc_pool.fill(mc, mc_pool_size)

    @Request.application
    def my_app(request):
        with mc_pool.reserve() as mc:
            key = request.path[1:].encode("ascii")
            val = mc.get(key)
            if not val:
                return NotFound(key)
            return Response(pformat(val))

    if __name__ == "__main__":
        from werkzeug.serving import run_simple
        run_simple("0.0.0.0", 5050, my_app)

I don't know if I think the above methodology is the best one though, another
possibility is to have a dict with thread names as keys and client objects for
values, then, each thread would look up its own client object in the dict on
each request, and if none exists, it clones a master just like the pooling
thing above. This would have to be a sort of LRU cache though, to avoid many
useless connections -- also would require some sort of garbage reuse mechanism,
complicated... Threads, I digress.

`pylibmc` has a helper class for this thread-mapped approach as well, but it's
a little broken in that it won't work very well when the average thread
lifespan is short compared to process lifetime: it wastes connections.
