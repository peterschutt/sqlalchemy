.. _asyncio_toplevel:

Asynchronous I/O (asyncio)
==========================

Support for Python asyncio.    Support for Core and ORM usage is
included, using asyncio-compatible dialects.

.. versionadded:: 1.4

.. warning:: Please read :ref:`asyncio_install` for important platform
   installation notes for many platforms, including **Apple M1 Architecture**.

.. tip:: The asyncio extension as of SQLAlchemy 1.4.3 can now be considered to
   be **beta level** software. API details are subject to change however at this
   point it is unlikely for there to be significant backwards-incompatible
   changes.

.. seealso::

    :ref:`change_3414` - initial feature announcement

    :ref:`examples_asyncio` - example scripts illustrating working examples
    of Core and ORM use within the asyncio extension.

.. _asyncio_install:

Asyncio Platform Installation Notes (Including Apple M1)
---------------------------------------------------------

The asyncio extension requires Python 3 only. It also depends
upon the `greenlet <https://pypi.org/project/greenlet/>`_ library. This
dependency is installed by default on common machine platforms including:

.. sourcecode:: text

    x86_64 aarch64 ppc64le amd64 win32

For the above platforms, ``greenlet`` is known to supply pre-built wheel files.
For other platforms, **greenlet does not install by default**;
the current file listing for greenlet can be seen at
`Greenlet - Download Files <https://pypi.org/project/greenlet/#files>`_.
Note that **there are many architectures omitted, including Apple M1**.

To install SQLAlchemy while ensuring the ``greenlet`` dependency is present
regardless of what platform is in use, the
``[asyncio]`` `setuptools extra <https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras>`_
may be installed
as follows, which will include also instruct ``pip`` to install ``greenlet``:

.. sourcecode:: text

  pip install sqlalchemy[asyncio]

Note that installation of ``greenlet`` on platforms that do not have a pre-built
wheel file means that ``greenlet`` will be built from source, which requires
that Python's development libraries also be present.


Synopsis - Core
---------------

For Core use, the :func:`_asyncio.create_async_engine` function creates an
instance of :class:`_asyncio.AsyncEngine` which then offers an async version of
the traditional :class:`_engine.Engine` API.   The
:class:`_asyncio.AsyncEngine` delivers an :class:`_asyncio.AsyncConnection` via
its :meth:`_asyncio.AsyncEngine.connect` and :meth:`_asyncio.AsyncEngine.begin`
methods which both deliver asynchronous context managers.   The
:class:`_asyncio.AsyncConnection` can then invoke statements using either the
:meth:`_asyncio.AsyncConnection.execute` method to deliver a buffered
:class:`_engine.Result`, or the :meth:`_asyncio.AsyncConnection.stream` method
to deliver a streaming server-side :class:`_asyncio.AsyncResult`::

    import asyncio

    from sqlalchemy.ext.asyncio import create_async_engine


    async def async_main():
        engine = create_async_engine(
            "postgresql+asyncpg://scott:tiger@localhost/test",
            echo=True,
        )

        async with engine.begin() as conn:
            await conn.run_sync(meta.drop_all)
            await conn.run_sync(meta.create_all)

            await conn.execute(
                t1.insert(), [{"name": "some name 1"}, {"name": "some name 2"}]
            )

        async with engine.connect() as conn:
            # select a Result, which will be delivered with buffered
            # results
            result = await conn.execute(select(t1).where(t1.c.name == "some name 1"))

            print(result.fetchall())

        # for AsyncEngine created in function scope, close and
        # clean-up pooled connections
        await engine.dispose()


    asyncio.run(async_main())

Above, the :meth:`_asyncio.AsyncConnection.run_sync` method may be used to
invoke special DDL functions such as :meth:`_schema.MetaData.create_all` that
don't include an awaitable hook.

.. tip:: It's advisable to invoke the :meth:`_asyncio.AsyncEngine.dispose` method
   using ``await`` when using the :class:`_asyncio.AsyncEngine` object in a
   scope that will go out of context and be garbage collected, as illustrated in the
   ``async_main`` function in the above example.  This ensures that any
   connections held open by the connection pool will be properly disposed
   within an awaitable context.   Unlike when using blocking IO, SQLAlchemy
   cannot properly dispose of these connections within methods like ``__del__``
   or weakref finalizers as there is no opportunity to invoke ``await``.
   Failing to explicitly dispose of the engine when it falls out of scope
   may result in warnings emitted to standard out resembling the form
   ``RuntimeError: Event loop is closed`` within garbage collection.

The :class:`_asyncio.AsyncConnection` also features a "streaming" API via
the :meth:`_asyncio.AsyncConnection.stream` method that returns an
:class:`_asyncio.AsyncResult` object.  This result object uses a server-side
cursor and provides an async/await API, such as an async iterator::

    async with engine.connect() as conn:
        async_result = await conn.stream(select(t1))

        async for row in async_result:
            print("row: %s" % (row,))

.. _asyncio_orm:


Synopsis - ORM
---------------

Using :term:`2.0 style` querying, the :class:`_asyncio.AsyncSession` class
provides full ORM functionality. Within the default mode of use, special care
must be taken to avoid :term:`lazy loading` or other expired-attribute access
involving ORM relationships and column attributes; the next
section :ref:`asyncio_orm_avoid_lazyloads` details this.   The example below
illustrates a complete example including mapper and session configuration::

    import asyncio

    from sqlalchemy import DateTime
    from sqlalchemy import ForeignKey
    from sqlalchemy import func
    from sqlalchemy import Integer
    from sqlalchemy import select
    from sqlalchemy import String
    from sqlalchemy.ext.asyncio import async_sessionmaker
    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship
    from sqlalchemy.orm import selectinload


    class Base(DeclarativeBase):
        pass


    class A(Base):
        __tablename__ = "a"

        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String)
        create_date = mapped_column(DateTime, server_default=func.now())
        bs = relationship("B")

        # required in order to access columns with server defaults
        # or SQL expression defaults, subsequent to a flush, without
        # triggering an expired load
        __mapper_args__ = {"eager_defaults": True}


    class B(Base):
        __tablename__ = "b"
        id = mapped_column(Integer, primary_key=True)
        a_id = mapped_column(ForeignKey("a.id"))
        data = mapped_column(String)


    async def async_main():
        engine = create_async_engine(
            "postgresql+asyncpg://scott:tiger@localhost/test",
            echo=True,
        )

        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.drop_all)
            await conn.run_sync(Base.metadata.create_all)

        # expire_on_commit=False will prevent attributes from being expired
        # after commit.
        async_session = async_sessionmaker(engine, expire_on_commit=False)

        async with async_session() as session:
            async with session.begin():
                session.add_all(
                    [
                        A(bs=[B(), B()], data="a1"),
                        A(bs=[B()], data="a2"),
                        A(bs=[B(), B()], data="a3"),
                    ]
                )

            stmt = select(A).options(selectinload(A.bs))

            result = await session.execute(stmt)

            for a1 in result.scalars():
                print(a1)
                print(f"created at: {a1.create_date}")
                for b1 in a1.bs:
                    print(b1)

            result = await session.execute(select(A).order_by(A.id))

            a1 = result.scalars().first()

            a1.data = "new data"

            await session.commit()

            # access attribute subsequent to commit; this is what
            # expire_on_commit=False allows
            print(a1.data)

        # for AsyncEngine created in function scope, close and
        # clean-up pooled connections
        await engine.dispose()


    asyncio.run(async_main())

In the example above, the :class:`_asyncio.AsyncSession` is instantiated using
the optional :class:`_asyncio.async_sessionmaker` helper, and associated with an
:class:`_asyncio.AsyncEngine` against particular database URL. It is
then used in a Python asynchronous context manager (i.e. ``async with:``
statement) so that it is automatically closed at the end of the block; this is
equivalent to calling the :meth:`_asyncio.AsyncSession.close` method.


.. _asyncio_orm_avoid_lazyloads:

Preventing Implicit IO when Using AsyncSession
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Using traditional asyncio, the application needs to avoid any points at which
IO-on-attribute access may occur. Above, the following measures are taken to
prevent this:

* The :func:`_orm.selectinload` eager loader is employed in order to eagerly
  load the ``A.bs`` collection within the scope of the
  ``await session.execute()`` call::

      stmt = select(A).options(selectinload(A.bs))

  ..

  If the default loader strategy of "lazyload" were left in place, the access
  of the ``A.bs`` attribute would raise an asyncio exception.
  There are a variety of ORM loader options available, which may be configured
  at the default mapping level or used on a per-query basis, documented at
  :ref:`loading_toplevel`.


* The :class:`_asyncio.AsyncSession` is configured using
  :paramref:`_orm.Session.expire_on_commit` set to False, so that we may access
  attributes on an object subsequent to a call to
  :meth:`_asyncio.AsyncSession.commit`, as in the line at the end where we
  access an attribute::

      # create AsyncSession with expire_on_commit=False
      async_session = AsyncSession(engine, expire_on_commit=False)

      # sessionmaker version
      async_session = async_sessionmaker(engine, expire_on_commit=False)

      async with async_session() as session:
          result = await session.execute(select(A).order_by(A.id))

          a1 = result.scalars().first()

          # commit would normally expire all attributes
          await session.commit()

          # access attribute subsequent to commit; this is what
          # expire_on_commit=False allows
          print(a1.data)

* The :paramref:`_schema.Column.server_default` value on the ``created_at``
  column will not be refreshed by default after an INSERT; instead, it is
  normally
  :ref:`expired so that it can be loaded when needed <orm_server_defaults>`.
  Similar behavior applies to a column where the
  :paramref:`_schema.Column.default` parameter is assigned to a SQL expression
  object. To access this value with asyncio, it has to be refreshed within the
  flush process, which is achieved by setting the
  :paramref:`_orm.Mapper.eager_defaults` parameter on the mapping::


    class A(Base):
        # ...

        # column with a server_default, or SQL expression default
        create_date = mapped_column(DateTime, server_default=func.now())

        # add this so that it can be accessed
        __mapper_args__ = {"eager_defaults": True}

Other guidelines include:

* Methods like :meth:`_asyncio.AsyncSession.expire` should be avoided in favor of
  :meth:`_asyncio.AsyncSession.refresh`

* Avoid using the ``all`` cascade option documented at :ref:`unitofwork_cascades`
  in favor of listing out the desired cascade features explicitly.   The
  ``all`` cascade option implies among others the :ref:`cascade_refresh_expire`
  setting, which means that the :meth:`.AsyncSession.refresh` method will
  expire the attributes on related objects, but not necessarily refresh those
  related objects assuming eager loading is not configured within the
  :func:`_orm.relationship`, leaving them in an expired state.

* Appropriate loader options should be employed for :func:`_orm.deferred`
  columns, if used at all, in addition to that of :func:`_orm.relationship`
  constructs as noted above.  See :ref:`orm_queryguide_column_deferral` for
  background on deferred column loading.

.. _dynamic_asyncio:

* The "dynamic" relationship loader strategy described at
  :ref:`dynamic_relationship` is not compatible by default with the asyncio approach.
  It can be used directly only if invoked within the
  :meth:`_asyncio.AsyncSession.run_sync` method described at
  :ref:`session_run_sync`, or by using its ``.statement`` attribute
  to obtain a normal select::

      user = await session.get(User, 42)
      addresses = (await session.scalars(user.addresses.statement)).all()
      stmt = user.addresses.statement.where(Address.email_address.startswith("patrick"))
      addresses_filter = (await session.scalars(stmt)).all()

  The :ref:`write only <write_only_relationship>` technique, introduced in
  version 2.0 of SQLAlchemy, is fully compatible with asyncio and should be
  preferred.

  .. seealso::

    :ref:`migration_20_dynamic_loaders` - notes on migration to 2.0 style

.. _session_run_sync:

Running Synchronous Methods and Functions under asyncio
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. deepalchemy::  This approach is essentially exposing publicly the
   mechanism by which SQLAlchemy is able to provide the asyncio interface
   in the first place.   While there is no technical issue with doing so, overall
   the approach can probably be considered "controversial" as it works against
   some of the central philosophies of the asyncio programming model, which
   is essentially that any programming statement that can potentially result
   in IO being invoked **must** have an ``await`` call, lest the program
   does not make it explicitly clear every line at which IO may occur.
   This approach does not change that general idea, except that it allows
   a series of synchronous IO instructions to be exempted from this rule
   within the scope of a function call, essentially bundled up into a single
   awaitable.

As an alternative means of integrating traditional SQLAlchemy "lazy loading"
within an asyncio event loop, an **optional** method known as
:meth:`_asyncio.AsyncSession.run_sync` is provided which will run any
Python function inside of a greenlet, where traditional synchronous
programming concepts will be translated to use ``await`` when they reach the
database driver.   A hypothetical approach here is an asyncio-oriented
application can package up database-related methods into functions that are
invoked using :meth:`_asyncio.AsyncSession.run_sync`.

Altering the above example, if we didn't use :func:`_orm.selectinload`
for the ``A.bs`` collection, we could accomplish our treatment of these
attribute accesses within a separate function::

    import asyncio

    from sqlalchemy import select
    from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine


    def fetch_and_update_objects(session):
        """run traditional sync-style ORM code in a function that will be
        invoked within an awaitable.

        """

        # the session object here is a traditional ORM Session.
        # all features are available here including legacy Query use.

        stmt = select(A)

        result = session.execute(stmt)
        for a1 in result.scalars():
            print(a1)

            # lazy loads
            for b1 in a1.bs:
                print(b1)

        # legacy Query use
        a1 = session.query(A).order_by(A.id).first()

        a1.data = "new data"


    async def async_main():
        engine = create_async_engine(
            "postgresql+asyncpg://scott:tiger@localhost/test",
            echo=True,
        )
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.drop_all)
            await conn.run_sync(Base.metadata.create_all)

        async with AsyncSession(engine) as session:
            async with session.begin():
                session.add_all(
                    [
                        A(bs=[B(), B()], data="a1"),
                        A(bs=[B()], data="a2"),
                        A(bs=[B(), B()], data="a3"),
                    ]
                )

            await session.run_sync(fetch_and_update_objects)

            await session.commit()

        # for AsyncEngine created in function scope, close and
        # clean-up pooled connections
        await engine.dispose()


    asyncio.run(async_main())

The above approach of running certain functions within a "sync" runner
has some parallels to an application that runs a SQLAlchemy application
on top of an event-based programming library such as ``gevent``.  The
differences are as follows:

1. unlike when using ``gevent``, we can continue to use the standard Python
   asyncio event loop, or any custom event loop, without the need to integrate
   into the ``gevent`` event loop.

2. There is no "monkeypatching" whatsoever.   The above example makes use of
   a real asyncio driver and the underlying SQLAlchemy connection pool is also
   using the Python built-in ``asyncio.Queue`` for pooling connections.

3. The program can freely switch between async/await code and contained
   functions that use sync code with virtually no performance penalty.  There
   is no "thread executor" or any additional waiters or synchronization in use.

4. The underlying network drivers are also using pure Python asyncio
   concepts, no third party networking libraries as ``gevent`` and ``eventlet``
   provides are in use.

.. _asyncio_events:

Using events with the asyncio extension
---------------------------------------

The SQLAlchemy :ref:`event system <event_toplevel>` is not directly exposed
by the asyncio extension, meaning there is not yet an "async" version of a
SQLAlchemy event handler.

However, as the asyncio extension surrounds the usual synchronous SQLAlchemy
API, regular "synchronous" style event handlers are freely available as they
would be if asyncio were not used.

As detailed below, there are two current strategies to register events given
asyncio-facing APIs:

* Events can be registered at the instance level (e.g. a specific
  :class:`_asyncio.AsyncEngine` instance) by associating the event with the
  ``sync`` attribute that refers to the proxied object. For example to register
  the :meth:`_events.PoolEvents.connect` event against an
  :class:`_asyncio.AsyncEngine` instance, use its
  :attr:`_asyncio.AsyncEngine.sync_engine` attribute as target. Targets
  include:

      :attr:`_asyncio.AsyncEngine.sync_engine`

      :attr:`_asyncio.AsyncConnection.sync_connection`

      :attr:`_asyncio.AsyncConnection.sync_engine`

      :attr:`_asyncio.AsyncSession.sync_session`

* To register an event at the class level, targeting all instances of the same type (e.g.
  all :class:`_asyncio.AsyncSession` instances), use the corresponding
  sync-style class. For example to register the
  :meth:`_ormevents.SessionEvents.before_commit` event against the
  :class:`_asyncio.AsyncSession` class, use the :class:`_orm.Session` class as
  the target.

When working within an event handler that is within an asyncio context, objects
like the :class:`_engine.Connection` continue to work in their usual
"synchronous" way without requiring ``await`` or ``async`` usage; when messages
are ultimately received by the asyncio database adapter, the calling style is
transparently adapted back into the asyncio calling style.  For events that
are passed a DBAPI level connection, such as :meth:`_events.PoolEvents.connect`,
the object is a :term:`pep-249` compliant "connection" object which will adapt
sync-style calls into the asyncio driver.

Some examples of sync style event handlers associated with async-facing API
constructs are illustrated below::

    import asyncio

    from sqlalchemy import event, text
    from sqlalchemy.engine import Engine
    from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
    from sqlalchemy.orm import Session

    ## Core events ##

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")


    # connect event on instance of Engine
    @event.listens_for(engine.sync_engine, "connect")
    def my_on_connect(dbapi_con, connection_record):
        print("New DBAPI connection:", dbapi_con)
        cursor = dbapi_con.cursor()

        # sync style API use for adapted DBAPI connection / cursor
        cursor.execute("select 'execute from event'")
        print(cursor.fetchone()[0])


    # before_execute event on all Engine instances
    @event.listens_for(Engine, "before_execute")
    def my_before_execute(
        conn,
        clauseelement,
        multiparams,
        params,
        execution_options,
    ):
        print("before execute!")


    ## ORM events ##

    session = AsyncSession(engine)


    # before_commit event on instance of Session
    @event.listens_for(session.sync_session, "before_commit")
    def my_before_commit(session):
        print("before commit!")

        # sync style API use on Session
        connection = session.connection()

        # sync style API use on Connection
        result = connection.execute(text("select 'execute from event'"))
        print(result.first())


    # after_commit event on all Session instances
    @event.listens_for(Session, "after_commit")
    def my_after_commit(session):
        print("after commit!")


    async def go():
        await session.execute(text("select 1"))
        await session.commit()

        await session.close()
        await engine.dispose()


    asyncio.run(go())

The above example prints something along the lines of:

.. sourcecode:: text

    New DBAPI connection: <AdaptedConnection <asyncpg.connection.Connection ...>>
    execute from event
    before execute!
    before commit!
    execute from event
    after commit!

.. topic:: asyncio and events, two opposites

    SQLAlchemy events by their nature take place within the **interior** of a
    particular SQLAlchemy process; that is, an event always occurs *after* some
    particular SQLAlchemy API has been invoked by end-user code, and *before*
    some other internal aspect of that API occurs.

    Contrast this to the architecture of the asyncio extension, which takes
    place on the **exterior** of SQLAlchemy's usual flow from end-user API to
    DBAPI function.

    The flow of messaging may be visualized as follows:

    .. sourcecode:: text

         SQLAlchemy    SQLAlchemy        SQLAlchemy          SQLAlchemy   plain
          asyncio      asyncio           ORM/Core            asyncio      asyncio
          (public      (internal)                            (internal)
          facing)
        -------------|------------|------------------------|-----------|------------
        asyncio API  |            |                        |           |
        call  ->     |            |                        |           |
                     |  ->  ->    |                        |  ->  ->   |
                     |~~~~~~~~~~~~| sync API call ->       |~~~~~~~~~~~|
                     | asyncio    |  event hooks ->        | sync      |
                     | to         |   invoke action ->     | to        |
                     | sync       |    event hooks ->      | asyncio   |
                     | (greenlet) |     dialect ->         | (leave    |
                     |~~~~~~~~~~~~|      event hooks ->    | greenlet) |
                     |  ->  ->    |       sync adapted     |~~~~~~~~~~~|
                     |            |               DBAPI -> |  ->  ->   | asyncio
                     |            |                        |           | driver -> database


    Where above, an API call always starts as asyncio, flows through the
    synchronous API, and ends as asyncio, before results are propagated through
    this same chain in the opposite direction. In between, the message is
    adapted first into sync-style API use, and then back out to async style.
    Event hooks then by their nature occur in the middle of the "sync-style API
    use".  From this it follows that the API presented within event hooks
    occurs inside the process by which asyncio API requests have been adapted
    to sync, and outgoing messages to the database API will be converted
    to asyncio transparently.

.. _asyncio_events_run_async:

Using awaitable-only driver methods in connection pool and other events
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As discussed in the above section, event handlers such as those oriented
around the :class:`.PoolEvents` event handlers receive a sync-style "DBAPI" connection,
which is a wrapper object supplied by SQLAlchemy asyncio dialects to adapt
the underlying asyncio "driver" connection into one that can be used by
SQLAlchemy's internals.    A special use case arises when the user-defined
implementation for such an event handler needs to make use of the
ultimate "driver" connection directly, using awaitable only methods on that
driver connection.  One such example is the ``.set_type_codec()`` method
supplied by the asyncpg driver.

To accommodate this use case, SQLAlchemy's :class:`.AdaptedConnection`
class provides a method :meth:`.AdaptedConnection.run_async` that allows
an awaitable function to be invoked within the "synchronous" context of
an event handler or other SQLAlchemy internal.  This method is directly
analogous to the :meth:`_asyncio.AsyncConnection.run_sync` method that
allows a sync-style method to run under async.

:meth:`.AdaptedConnection.run_async` should be passed a function that will
accept the innermost "driver" connection as a single argument, and return
an awaitable that will be invoked by the :meth:`.AdaptedConnection.run_async`
method.  The given function itself does not need to be declared as ``async``;
it's perfectly fine for it to be a Python ``lambda:``, as the return awaitable
value will be invoked after being returned::

    from sqlalchemy import event
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine(...)


    @event.listens_for(engine.sync_engine, "connect")
    def register_custom_types(dbapi_connection, *args):
        dbapi_connection.run_async(
            lambda connection: connection.set_type_codec(
                "MyCustomType",
                encoder,
                decoder,  # ...
            )
        )

Above, the object passed to the ``register_custom_types`` event handler
is an instance of :class:`.AdaptedConnection`, which provides a DBAPI-like
interface to an underlying async-only driver-level connection object.
The :meth:`.AdaptedConnection.run_async` method then provides access to an
awaitable environment where the underlying driver level connection may be
acted upon.

.. versionadded:: 1.4.30


Using multiple asyncio event loops
----------------------------------

An application that makes use of multiple event loops, for example in the
uncommon case of combining asyncio with multithreading, should not share the
same :class:`_asyncio.AsyncEngine` with different event loops when using the
default pool implementation.

If an :class:`_asyncio.AsyncEngine` is be passed from one event loop to another,
the method :meth:`_asyncio.AsyncEngine.dispose()` should be called before it's
re-used on a new event loop. Failing to do so may lead to a ``RuntimeError``
along the lines of
``Task <Task pending ...> got Future attached to a different loop``

If the same engine must be shared between different loop, it should be configured
to disable pooling using :class:`~sqlalchemy.pool.NullPool`, preventing the Engine
from using any connection more than once::

    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.pool import NullPool

    engine = create_async_engine(
        "postgresql+asyncpg://user:pass@host/dbname",
        poolclass=NullPool,
    )

.. _asyncio_scoped_session:

Using asyncio scoped session
----------------------------

The "scoped session" pattern used in threaded SQLAlchemy with the
:class:`.scoped_session` object is also available in asyncio, using
an adapted version called :class:`_asyncio.async_scoped_session`.

.. tip::  SQLAlchemy generally does not recommend the "scoped" pattern
   for new development as it relies upon mutable global state that must also be
   explicitly torn down when work within the thread or task is complete.
   Particularly when using asyncio, it's likely a better idea to pass the
   :class:`_asyncio.AsyncSession` directly to the awaitable functions that need
   it.

When using :class:`_asyncio.async_scoped_session`, as there's no "thread-local"
concept in the asyncio context, the "scopefunc" parameter must be provided to
the constructor. The example below illustrates using the
``asyncio.current_task()`` function for this purpose::

    from asyncio import current_task

    from sqlalchemy.ext.asyncio import (
        async_scoped_session,
        async_sessionmaker,
    )

    async_session_factory = async_sessionmaker(
        some_async_engine,
        expire_on_commit=False,
    )
    AsyncScopedSession = async_scoped_session(
        async_session_factory,
        scopefunc=current_task,
    )
    some_async_session = AsyncScopedSession()

.. warning:: The "scopefunc" used by :class:`_asyncio.async_scoped_session`
   is invoked **an arbitrary number of times** within a task, once for each
   time the underlying :class:`_asyncio.AsyncSession` is accessed. The function
   should therefore be **idempotent** and lightweight, and should not attempt
   to create or mutate any state, such as establishing callbacks, etc.

.. warning:: Using ``current_task()`` for the "key" in the scope requires that
   the :meth:`_asyncio.async_scoped_session.remove` method is called from
   within the outermost awaitable, to ensure the key is removed from the
   registry when the task is complete, otherwise the task handle as well as
   the :class:`_asyncio.AsyncSession` will remain in memory, essentially
   creating a memory leak.  See the following example which illustrates
   the correct use of :meth:`_asyncio.async_scoped_session.remove`.

:class:`_asyncio.async_scoped_session` includes **proxy
behavior** similar to that of :class:`.scoped_session`, which means it can be
treated as a :class:`_asyncio.AsyncSession` directly, keeping in mind that
the usual ``await`` keywords are necessary, including for the
:meth:`_asyncio.async_scoped_session.remove` method::

    async def some_function(some_async_session, some_object):
        # use the AsyncSession directly
        some_async_session.add(some_object)

        # use the AsyncSession via the context-local proxy
        await AsyncScopedSession.commit()

        # "remove" the current proxied AsyncSession for the local context
        await AsyncScopedSession.remove()

.. versionadded:: 1.4.19

.. currentmodule:: sqlalchemy.ext.asyncio


.. _asyncio_inspector:

Using the Inspector to inspect schema objects
---------------------------------------------------

SQLAlchemy does not yet offer an asyncio version of the
:class:`_reflection.Inspector` (introduced at :ref:`metadata_reflection_inspector`),
however the existing interface may be used in an asyncio context by
leveraging the :meth:`_asyncio.AsyncConnection.run_sync` method of
:class:`_asyncio.AsyncConnection`::

    import asyncio

    from sqlalchemy import inspect
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost/test")


    def use_inspector(conn):
        inspector = inspect(conn)
        # use the inspector
        print(inspector.get_view_names())
        # return any value to the caller
        return inspector.get_table_names()


    async def async_main():
        async with engine.connect() as conn:
            tables = await conn.run_sync(use_inspector)


    asyncio.run(async_main())

.. seealso::

    :ref:`metadata_reflection`

    :ref:`inspection_toplevel`

Engine API Documentation
-------------------------

.. autofunction:: create_async_engine

.. autofunction:: async_engine_from_config

.. autoclass:: AsyncEngine
   :members:

.. autoclass:: AsyncConnection
   :members:

.. autoclass:: AsyncTransaction
   :members:

Result Set API Documentation
----------------------------------

The :class:`_asyncio.AsyncResult` object is an async-adapted version of the
:class:`_result.Result` object.  It is only returned when using the
:meth:`_asyncio.AsyncConnection.stream` or :meth:`_asyncio.AsyncSession.stream`
methods, which return a result object that is on top of an active database
cursor.

.. autoclass:: AsyncResult
   :members:

.. autoclass:: AsyncScalarResult
   :members:

.. autoclass:: AsyncMappingResult
   :members:

ORM Session API Documentation
-----------------------------

.. autofunction:: async_object_session

.. autofunction:: async_session

.. autoclass:: async_sessionmaker
   :members:
   :inherited-members:

.. autoclass:: async_scoped_session
   :members:
   :inherited-members:

.. autoclass:: AsyncSession
   :members:
   :exclude-members: sync_session_class

   .. autoattribute:: sync_session_class

.. autoclass:: AsyncSessionTransaction
   :members:



