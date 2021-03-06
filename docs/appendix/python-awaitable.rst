.. meta::
    :keywords: Python, Python3, coroutine, asyncio, awaitable

==========================================
Asynochronous Programming: To Be Continued
==========================================

.. contents:: Table of Contents
    :backlinks: none

Abstract
--------

Similar concepts

1. Javascript: Promise/resolve/reject/then
2. C++11: promise/promise::set_value/promise::set_exception/future::get
3. Python: Future/Future.set_result/Future.set_exception/Future.result

.. code-block:: javascript

    const https = require('https');

    let req = https.get('https://github.com', res => {
      res.on('data', d => console.log(d.toString('utf8')));
    })
    req.on('error', e => console.log(e));
    req.end();

.. code-block:: javascript

    const https = require('https');

    function httpGet(url) {
      return new Promise((resolve, reject) => {
        let req = https.get(url, res => {
          let buf = [];
          res.on('data', d => buf.push(d));
          res.on('end', () => resolve(buf.toString('utf8')));
        })
        req.on('error', e => reject(e));
        req.end();
      });
    }

    httpGet('https://github.com')
      .then(d => console.log(d))
      .catch(e => console.error(e));


.. code-block:: python3

    import asyncio
    import aiohttp

    async def fetch(session, url):
        async with session.get(url) as response:
            return await response.text()

    async def http_get(url, handler):
        async with aiohttp.ClientSession() as session:
            try:
                handler(await fetch(session, url))
            except Exception as e:
                handler(None, e)

    def handler(data, err=None):
        if err:
            raise err
        print(data)

    asyncio.run(http_get("https://github.com", handler))

What is Task?
--------------

.. code-block:: python

    # goal: supervise coroutine run state
    # ref: asyncio/tasks.py

    import asyncio
    Future = asyncio.futures.Future

    class Task(Future):
        """Simple prototype of Task"""

        def __init__(self, gen, *, loop):
            super().__init__(loop=loop)
            self._gen = gen
            self._loop.call_soon(self._step)

        def _step(self, val=None, exc=None):
            try:
                if exc:
                    f = self._gen.throw(exc)
                else:
                    f = self._gen.send(val)
            except StopIteration as e:
                self.set_result(e.value)
            except Exception as e:
                self.set_exception(e)
            else:
                f.add_done_callback(
                     self._wakeup)

        def _wakeup(self, fut):
            try:
                res = fut.result()
            except Exception as e:
                self._step(None, e)
            else:
                self._step(res, None)

    @asyncio.coroutine
    def foo():
        yield from asyncio.sleep(3)
        print("Hello Foo")

    @asyncio.coroutine
    def bar():
        yield from asyncio.sleep(1)
        print("Hello Bar")

    loop = asyncio.get_event_loop()
    tasks = [Task(foo(), loop=loop),
             loop.create_task(bar())]
    loop.run_until_complete(
            asyncio.wait(tasks))
    loop.close()

output:

.. code-block:: console

    $ python test.py
    Hello Bar
    hello Foo

How does event loop work?
-------------------------

.. code-block:: python

    import asyncio
    from collections import deque

    def done_callback(fut):
        fut._loop.stop()

    class Loop:
        """Simple event loop prototype"""

        def __init__(self):
            self._ready = deque()
            self._stopping = False

        def create_task(self, coro):
            Task = asyncio.tasks.Task
            task = Task(coro, loop=self)
            return task

        def run_until_complete(self, fut):
            tasks = asyncio.tasks
            # get task
            fut = tasks.ensure_future(
                        fut, loop=self)
            # add task to ready queue
            fut.add_done_callback(done_callback)
            # run tasks
            self.run_forever()
            # remove task from ready queue
            fut.remove_done_callback(done_callback)

        def run_forever(self):
            """Run tasks until stop"""
            try:
                while True:
                    self._run_once()
                    if self._stopping:
                        break
            finally:
                self._stopping = False

        def call_soon(self, cb, *args):
            """Append task to ready queue"""
            self._ready.append((cb, args))
        def call_exception_handler(self, c):
            pass

        def _run_once(self):
            """Run task at once"""
            ntodo = len(self._ready)
            for i in range(ntodo):
                t, a = self._ready.popleft()
                t(*a)

        def stop(self):
            self._stopping = True

        def close(self):
            self._ready.clear()

        def get_debug(self):
            return False

    @asyncio.coroutine
    def foo():
        print("Foo")

    @asyncio.coroutine
    def bar():
        print("Bar")

    loop = Loop()
    tasks = [loop.create_task(foo()),
             loop.create_task(bar())]
    loop.run_until_complete(
            asyncio.wait(tasks))
    loop.close()

output:

.. code-block:: console

    $ python test.py
    Foo
    Bar

Simple asyncio.run
-------------------

.. code-block:: python

    >>> import asyncio
    >>> async def getaddrinfo(host, port):
    ...     loop = asyncio.get_event_loop()
    ...     return (await loop.getaddrinfo(host, port))
    ...
    >>> def run(main):
    ...     loop = asyncio.new_event_loop()
    ...     asyncio.set_event_loop(loop)
    ...     return loop.run_until_complete(main)
    ...
    >>> ret = run(getaddrinfo('google.com', 443))
    >>> ret = asyncio.run(getaddrinfo('google.com', 443))
