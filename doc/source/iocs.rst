*******************************
Input-Output Controllers (IOCs)
*******************************

EPICS Input-Output Controllers (IOCs) expose an EPICS server. Behind the
server, they may connect to a device driver, data-processing code, and/or
an EPICS client for chaining with other servers.

In this section, we will review some of the example IOCs that are included with
caproto, intended as demonstrations of what is possible.

Why Write an IOC Using Caproto?
===============================

Caproto makes it is easy to launch a protocol-compliant Channel Access server
in just a couple lines of Python. This opens up some interesting possibilities:

* In Python, it is easy to invoke standard web protocols. For example, writing
  an EPICS server around a device that speaks JSON may be easier with caproto
  than with any previously-existing tools.
* Many scientists who rely on EPICS but may not understand the details of
  EPICS already know some Python for the data analysis work. Caproto may make
  it easier for scientists and controls engineers to collaborate.
* As with its clients, caproto's servers handle a human-friendly encapsulation
  of every message sent and received, which can be valuable for development,
  logging, and debugging.

Take a look around
`the ioc_examples subpackage
<https://github.com/caproto/caproto/tree/master/caproto/ioc_examples>`_ for
more examples not covered in this document.

Using the IOC Examples
======================

.. currentmodule:: caproto

They can be started like so:

.. code-block:: bash

    $ python3 -m caproto.ioc_examples.simple
    [I 16:51:09.751 server:93] Server starting up...
    [I 16:51:09.752 server:109] Listening on 0.0.0.0:54966
    [I 16:51:09.753 server:121] Server startup complete.

and stopped using Ctrl+C:

.. code-block:: bash

    ^C
    [I 16:51:10.828 server:129] Server task cancelled. Must shut down.
    [I 16:51:10.828 server:132] Server exiting....

Use ``--list-pvs`` to display which PVs they serve:

.. code-block:: bash

    $ python3 -m caproto.ioc_examples.simple --list-pvs
    [I 16:52:36.087 server:93] Server starting up...
    [I 16:52:36.089 server:109] Listening on 0.0.0.0:62005
    [I 16:52:36.089 server:121] Server startup complete.
    [I 16:52:36.089 server:123] PVs available:
        simple:A
        simple:B
        simple:C

and use ``--prefix`` to conveniently customize the PV prefix:

.. code-block:: bash

    $ python3 -m caproto.ioc_examples.simple --list-pvs --prefix my_custom_prefix:
    [I 16:54:14.528 server:93] Server starting up...
    [I 16:54:14.530 server:109] Listening on 0.0.0.0:55810
    [I 16:54:14.530 server:121] Server startup complete.
    [I 16:54:14.530 server:123] PVs available:
        my_custom_prefix:A
        my_custom_prefix:B
        my_custom_prefix:C

Type ``python3 -m caproto.ioc_examples.simple -h`` for more options.

PVGroup
=======

The :class:`~.server.PVGroup` is a container for ``pvproperty``
instances which determine how to represent the group's contents as EPICS
process variables (PVs).

You are expected to subclass from ``PVGroup`` to implement your own IOCs.

.. code:: python

    from caproto.server import PVGroup, pvproperty

    class BasicIOC(PVGroup):
        my_pv = pvproperty(value=0, doc="My test PV")


caproto uses your class definition and performs magic under the hood to create
a simple PV name to :class:`~.server.ChannelData` instances.  See what
happens when you instantiate the above:

.. code:: ipython

    In [1]: ioc = BasicIOC(prefix='prefix:')

    In [2]: dict(ioc.pvdb)
    Out[2]: {'prefix:my_pv': <caproto.server.server.PvpropertyInteger at 0x7fe5c069f910>}


This simple dictionary database is what will be used to start your server.
Adding on some boilerplate to help you parse command-line arguments and pick
an async library backend to use:

.. code:: python

    from caproto.server import ioc_arg_parser, run
    ioc_options, run_options = ioc_arg_parser(
        default_prefix='prefix:',
        desc="Basic IOC",
    )
    ioc = BasicIOC(**ioc_options)
    run(ioc.pvdb, **run_options)

The above would allow you to access ``prefix:my_pv`` over Channel Access.

group_write
-----------

By default, if an explicit putter hook is not supplied for a given
`.pvproperty`, an implicit one defined in the group will be used.

.. code:: python

    class MyGroup(PVGroup):
        async def group_write(self, instance, value):
            """Generic write called for channels without `put` defined."""
            print(f"{instance.pvspec.attr} was written to with {value}.")


This can be used to generically handle or modify the put handling of
any pvproperty in the group.

Alarms
------

PVGroup by default has an ``alarms`` attribute which is defined as follows:

.. code:: python

    alarms = collections.defaultdict(ChannelAlarm)

Individual ``pvproperty`` instances specify their alarm instance as keys of
this dictionary.  That is,

.. code:: python

    my_property = pvproperty(value=0, alarm_group="primary")

means that ``my_property`` will join the "primary" alarm group, or the
instance from

.. code:: python

    pvgroup_instance.alarms["primary"]

While this is not required to be a string, it is recommended to do so by
convention.

Note that when ``alarm_group`` is unspecified, pvproperties in a given PVGroup
instance share the same alarm instance - as their key to the above dictionary
is left at the default value. This is usually a reasonable guess for small
groups of PVs.

API
---

.. autosummary::
    :toctree: generated

    caproto.server.PVGroup


pvproperty
==========

``pvproperty`` combines information as to how to represent your information
over EPICS Channel Access and provides easy-to-use hooks for reacting to
certain events -- by hooking user-provided Python methods.

Supported data types
--------------------

``value`` may be an instance of one of the following, or ``dtype`` may be
one of the following types:

.. list-table:: Built-in Data Type Mapping
   :header-rows: 1

   * - Data Type
     - Data Class
     - Inherits from
   * - str
     - :class:`~.server.PvpropertyChar`
     - :class:`~.ChannelChar`
   * - bytes
     - :class:`~.server.PvpropertyByte`
     - :class:`.ChannelByte`
   * - int
     - :class:`~.server.PvpropertyInteger`
     - :class:`.ChannelInteger`
   * - float
     - :class:`~.server.PvpropertyDouble`
     - :class:`.ChannelDouble`
   * - bool
     - :class:`~.server.PvpropertyBoolEnum`
     - :class:`.ChannelEnum`
   * - enum.IntEnum
     - :class:`~.server.PvpropertyEnum`
     - :class:`.ChannelEnum`

``dtype`` may be omitted when a value is specified.

If you desire finer-grained control over the data type provided (e.g., you
specifically want a 32-bit float instead of a 64-bit double), you may
use the appropriate `ChannelType` directly:

.. list-table:: ChannelType Mapping
   :header-rows: 1

   * - Data Type
     - Data Class
     - Inherits from
   * - ChannelType.STRING
     - :class:`~.server.PvpropertyString`
     - :class:`~.ChannelString`
   * - ChannelType.INT
     - :class:`~.server.PvpropertyShort`
     - :class:`~.ChannelShort`
   * - ChannelType.LONG
     - :class:`~.server.PvpropertyInteger`
     - :class:`~.ChannelInteger`
   * - ChannelType.DOUBLE
     - :class:`~.server.PvpropertyDouble`
     - :class:`~.ChannelDouble`
   * - ChannelType.FLOAT
     - :class:`~.server.PvpropertyFloat`
     - :class:`~.ChannelFloat`
   * - ChannelType.ENUM
     - :class:`~.server.PvpropertyEnum`
     - :class:`~.ChannelEnum`
   * - ChannelType.CHAR
     - :class:`~.server.PvpropertyChar`
     - :class:`~.ChannelChar`

These are the special classes pvproperty uses:

.. autosummary::
    :toctree: generated

    caproto.server.PvpropertyBoolEnum
    caproto.server.PvpropertyByte
    caproto.server.PvpropertyChar
    caproto.server.PvpropertyDouble
    caproto.server.PvpropertyEnum
    caproto.server.PvpropertyFloat
    caproto.server.PvpropertyInteger
    caproto.server.PvpropertyShort
    caproto.server.PvpropertyString


The following are the underlying, lowest-level classes for holding data which
can be sent over an EPICS V3 channel in caproto:

.. autosummary::
    :toctree: generated

    caproto.ChannelByte
    caproto.ChannelChar
    caproto.ChannelDouble
    caproto.ChannelEnum
    caproto.ChannelFloat
    caproto.ChannelInteger
    caproto.ChannelShort
    caproto.ChannelString


Integers and floats
^^^^^^^^^^^^^^^^^^^

Integer and floating point values by default map to the largest containers
in EPICS - 32-bit integers and 64-bit floats, respectively.

Strings
^^^^^^^

Strings are a bit complicated when it comes to EPICS and caproto follows suit.
See the "how to" section below for more information on long strings.


Enums
^^^^^

A new syntax for specifying enums using built-in :class:`enum.IntEnum` is
recommended:

.. code:: python


    from caproto.server import PVGroup, pvproperty
    import enum


    class MyEnum(enum.IntEnum):
        off = 0
        on = 1
        unknown = 2


    class MyPVGroup(PVGroup):
        my_enum = pvproperty(
            value=MyEnum.on,
            record='mbbi',
            doc="An enum with off/on/unknown, and mbbi record fields.",
        )

Alternatively, you may specify the strings manually:

.. code:: python

    from caproto import ChannelType
    from caproto.server import PVGroup, pvproperty

    class MyPVGroup(PVGroup):
        my_enum = pvproperty(
            value='on',
            enum_strings=("off", "on", "unknown"),
            dtype=ChannelType.ENUM,
            record='mbbi',
            doc="An enum with off/on/unknown, and mbbi record fields.",
        )

For the above mbbi records, the ``ZRST`` (zero string) field, ``ONST`` (one
string) field, and so on (up to 15), are similarly respected and mapped from
the enum strings (keyword argument or ``IntEnum`` class).


Booleans
^^^^^^^^

Boolean values map to an enum type, with "Off" and "On" as the default
strings.

When using either the ``bi`` or ``bo`` records, the ``ZNAM`` and ``ONAM``
fields are automatically populated with the string equivalent values for 0 and
1.  These are derived from the ``enum_strings`` keyword argument.

Hooks
-----

Hooks allow you to react to certain events that happen during the life of
the IOC by providing your own methods.

Putter Hook
^^^^^^^^^^^

This is the most common hook you will need when writing a caproto-based IOC.

The "putter" associated with a ``pvproperty`` is called whenever a user writes
to the associated PV through any client (``caput``, ``caproto-put``, pyepics,
caproto clients, and so on) and also when you write directly to it in code.

An example of usage and the required method signature is as follows:

.. code:: python

    my_property = pvproperty(value=0)

    @my_property.putter
    async def my_property(self, instance, value):
        """
        Startup hook for ``my_property``.

        Parameters
        ----------
        instance : ChannelData
            This is the instance of ``my_property``.

        value :
            This is the value the client wrote to the PV.

        Returns
        -------
        value :
            You may optionally change the value before committing it into
            the underlying ChannelData instance.

        Raises
        ------
        SkipWrite
            Raise this to skip further processing of the write.

        Exception
            When an unhandled error occurs.  Alarm will be set.
        """
        ...


Note that in the above, ``self`` will refer to the ``PVGroup`` instance.
It will allow you to access the values of other PVs in the group, or any
other state held within.


``instance`` here is the ChannelData-based instance of ``my_property``.
It is exactly equivalent to ``self.my_property`` in the above example.  This
object holds the current value, alarm instance, and other associated metadata.

Since ``putter`` methods may be reused among any number of ``pvproperty``
instances, this parameter can be used generically in such scenarios.

The ``value`` can be assumed to be consistent with the data type of
``my_property``.  In the above case, this is ``int`` (as ``0`` is an integer).

You may optionally change the value before committing it into the underlying
ChannelData instance.  If this is needed, return the modified value here.
Otherwise, the implicit value of ``None`` means to accept the value as-is.

Additionally, you may completely reject the write entirely by raising the
`.SkipWrite` exception.  This is a non-error exception which will leave the
previously-stored value intact.


Startup Hook
^^^^^^^^^^^^

This hook is executed when the IOC starts up.

An example of usage and the required method signature is as follows:

.. code:: python

    @my_property.startup
    async def my_property(self, instance, async_lib):
        """
        Startup hook for ``my_property``.

        Parameters
        ----------
        instance : ChannelData
            This is the instance of ``my_property``.

        async_lib : AsyncLibraryLayer
            This is a shim layer for {asyncio, curio, trio} that you can use
            to make async library-agnostic IOCs.
        """
        ...


Note that in the above, ``self`` will refer to the ``PVGroup`` instance.
It will allow you to access the values of other PVs in the group, or any
other state held within.

Second, notice that ``my_property`` is repeated multiple times:  the decorator
must use ``my_property.startup`` to define the startup method, and the
method itself must be named ``my_property``.  This is due to how decorators
function and mirrors what you would see with a standard Python ``property``.

Finally, the ``async_lib`` can be used to abstract away the async library
layer that is in use.  It is considered good practice to use, for example,
``async_lib.sleep`` instead of ``asyncio.sleep`` -- or similarly
``trio.sleep``, ``curio.sleep``.  This will allow your IOC to work no matter if
you decide to use asyncio, curio, or trio as your async library.


Shutdown Hook
^^^^^^^^^^^^^

This hook is executed when the IOC shuts down.

An example of usage and the required method signature is as follows:

.. code:: python

    @my_property.shutdown
    async def my_property(self, instance, async_lib):
        """
        Shutdown hook for ``my_property``.

        Parameters
        ----------
        instance : ChannelData
            This is the instance of ``my_property``.

        async_lib : AsyncLibraryLayer
            This is a shim layer for {asyncio, curio, trio} that you can use
            to make async library-agnostic IOCs.
        """
        ...

See the startup hook section above for more details on what these parameters
mean.

Scan Hook
^^^^^^^^^

This hook is executed periodically, with the exact rate depending on its
configuration.

.. code:: python

    @my_property.scan(period=0.1)
    async def my_property(self, instance, async_lib):
        """
        Scan hook for ``my_property``.

        Parameters
        ----------
        instance : ChannelData
            This is the instance of ``my_property``.

        async_lib : AsyncLibraryLayer
            This is a shim layer for {asyncio, curio, trio} that you can use
            to make async library-agnostic IOCs.
        """
        ...

See the startup hook section above for more details on what these parameters
mean.

`.scan_wrapper` arguments can be passed to ``my_property.scan(...)`` above
to further customize how the routine is called.  For example, it is possible
to stop the scan routine when an exception happens (``stop_on_error=True``)
or let users put to the ``.SCAN`` field (when using ``record="..."``) to
change the rate to common EPICS-defined ones.


.. autosummary::
    :toctree: generated

    caproto.server.scan_wrapper


Records
-------

``pvproperty`` allows you to pretend your value has common EPICS record fields.
For example, it is possible to report a given property as an analog input
(``ai``) record:

.. code:: python

    my_property = pvproperty(value=1.0, record='ai')

More on this in a later section.

API
---

.. autosummary::
    :toctree: generated

    caproto.server.pvproperty


General Tips
============

Don't use a getter
------------------

Generally speaking, you should not define a getter. This goes back to original
design decisions for how ``pvproperty`` instances could work. The authors
now know that this was a mistake to make easily accessible.

Why?
^^^^

Inclusion of a getter can unfortunately make value monitoring less intuitive at
best, or break its functionality at worst.

There is no way to monitor such a value for changes, as it requires calling
the getter to determine the current value.

Each time a user runs ``caget YOUR:PV``, the getter will be called. If the
getter performs a write to update the instance, all other clients watching
``YOUR:PV`` will then see an update. The more clients added to the mix, the
more frequent that other clients will see updates - seemingly at arbitrary
times.

Reactive design
---------------

Most of what your IOC does should be in response to what the user requests by
way of ``putter`` hooks.

Periodic updates should happen in ``scan`` loops.

If PVGroup data is sourced from a database, real device, website, etc,
then a different approach may be more appropriate.  See below in the "how do I"
section.

How do I...
===========

... access ``async_lib`` directly in my class?
----------------------------------------------

If you have any startup hooks defined, you can stash ``async_lib`` there.

.. code:: python

    thing = pvproperty(value=2, doc="An integer-valued PV.")

    @thing.startup
    async def thing(self, instance, async_lib):
        self.async_lib = async_lib


Or more generally, you can pass a ``startup_hook`` independent of any
``pvproperty`` by way of ``run()``:

.. code:: python

    class StartupAndShutdown(PVGroup):
        async def __ainit__(self, async_lib):
            self.log.warning("1. The IOC-level startup_hook from `run()` was called.")
            # Note that we have to pass this in to ``run()``!

    if __name__ == '__main__':
        ioc_options, run_options = ioc_arg_parser(
            default_prefix='simple:',
            desc="Run an IOC that prints on startup and shutdown.")
        ioc = StartupAndShutdown(**ioc_options)
        run(ioc.pvdb, startup_hook=ioc.__ainit__, **run_options)

As an convention, consider naming your method ``__ainit__``.

See the full IOC example in ``startup_and_shutdown_hooks``.

... stop every PV in my group from sharing the same alarm?
----------------------------------------------------------

By default, pvproperties in a given PVGroup instance share the same alarm
instance. This is usually a reasonable guess for small groups of PVs.  For
finer-grained customization, the easiest way to change this functionality is to
specify a per-pvproperty alarm group using the ``alarm_group`` keyword
argument.  It expects a string identifier, which should be reused on all other
pvproperties that are expected to use the same alarm instance.

... make a string PV? All I see is an array of numbers!
-------------------------------------------------------

If you have a short string that can fit into the EPICS definition of
``DBR_STRING`` - a maximum length of ``MAX_STRING_SIZE = 40`` - you can use
the following and not worry about anything further:

.. code:: python

    my_property = pvproperty(
        # The default value:
        value="String value",
        # Specify STRING as the data type - otherwise CHAR will be chosen:
        dtype=ChannelType.STRING,
        # Document what the property is for:
        doc="Indicator as to what this does.",
        # Optionally specify the encoding, if necessary:
        string_encoding='latin-1',
    )

There is a convention you may not have heard about, what caproto refers to as
long string support, in which ``DBR_STRING`` values can be longer than 40
characters when **clients** append a special character to the PV name:

.. code:: bash

    $ caget -S my_property.$ my_property.VAL$
    my_property.$ String value
    my_property.VAL$ String value

    $ caput -S my_property.VAL$ 1234567890123456789012345678901234567890123456789012345678901234567890
    Old : my_property.VAL$ String value
    New my_property.VAL$ 1234567890123456789012345678901234567890123456789012345678901234567890


Note that the last string is 70 characters long.  What happens if we request it
as a regular string without the long string modifier now?

.. code:: bash

    $ caget my_property
    my_property       1234567890123456789012345678901234567890


... it's truncated to only 40 characters! There's not much we can do about it,
so clients should just be configured to use the custom modifier.

As an alternative, :class:`~.server.pvproperty` supports arrays of ``DBR_CHAR``
which are signed, 8-bit numbers, meaning its values fall in the inclusive range
of [-127, 127].  These can also be used to represent arbitrarily long strings.

.. code:: python

    my_property = pvproperty(
        # The default value:
        value="String value",
        # Document it!
        doc="Indicator as to what this does.",
        # Optionally configure the encoding:
        string_encoding='utf-8',
        # Ensure that this is marked as "report_as_string" (** optional)
        report_as_string=True,
        # Optionally specify how long the string can get - the default is
        # the length of the provided value.
        max_length=255,
    )


Please note that you will still have to use the special long string modifier
``.$`` to have this work with ``caget`` and ``caput``.

``report_as_string=True`` is optional here, but it makes it function more
closely to the first example provided in this section.  You can access up
to the first 40 characters of the string without any special modifiers to
``caget`` and ``caput``, or ``caproto-get`` and ``caproto-put``.

... structure an IOC when talking to a real piece of hardware?
--------------------------------------------------------------

When talking to a single device and requesting a bunch of different information
from it to update a single ``PVGroup`` to represent its status, we recommend
the inclusion of a polling loop.

For a simple "query device and update state"-style group, this could look like:

.. code:: python

    update_hook = pvproperty(name="update", value=False, record='bi')

    @update_hook.scan(period=0.1, use_scan_field=True)
    async def update_hook(self, instance, async_lib):
        """
        Scan hook for ``update_hook``.

        Parameters
        ----------
        instance : ChannelData
            This is the instance of ``update_hook``.

        async_lib : AsyncLibraryLayer
            This is a shim layer for {asyncio, curio, trio} that you can use
            to make async library-agnostic IOCs.
        """
        # Reach out to the device asynchronously and get back information:
        device_info = await query_device()
        await self.value1.write(value=device_info["value1"])
        await self.value2.write(value=device_info["value2"])
        await self.value3.write(value=device_info["value3"])


With the above, the client has some control over how fast the updates happen.
If that's undesirable, set ``period=desired_rate`` and ``use_scan_field=False``

If you don't have an async-capable interface, there will be some additional
work required.  Consider using async_lib queues for this.

.. code:: python

    update_hook = pvproperty(name="update", value=False, record='bi')

    @update_hook.startup
    async def update_hook(self, instance, async_lib):
        """
        Startup hook for ``update_hook``.

        Parameters
        ----------
        instance : ChannelData
            This is the instance of ``update_hook``.

        async_lib : AsyncLibraryLayer
            This is a shim layer for {asyncio, curio, trio} that you can use
            to make async library-agnostic IOCs.
        """
        queue = async_lib.ThreadsafeQueue()
        thread = threading.Thread(target=my_threaded_function, kwargs=dict(queue=queue))
        thread.start()

        try:
            while True:  # or perhaps thread.is_alive()
                value = await queue.async_get()
                await self.prop.write(value=value["prop"])
        finally:
            ...  # perform cleanup here


If, instead, you have a bunch of knobs that the user can set with some
control flow decisions to make, instead consider something like:

.. code:: python

    update_hook = pvproperty(name="update", value=False, record='bi')

    @update_hook.startup
    async def update_hook(self, instance, async_lib):
        """
        Startup hook for ``update_hook``.

        Parameters
        ----------
        instance : ChannelData
            This is the instance of ``update_hook``.

        async_lib : AsyncLibraryLayer
            This is a shim layer for {asyncio, curio, trio} that you can use
            to make async library-agnostic IOCs.
        """
        while True:
            device_info = await query_device()
            await self.value1.write(value=device_info["value1"])
            if self.user_requested.value == 'move':
                await self.queue_move()
            ...
            await async_lib.sleep(update_delay)


A more concrete example with aiohttp:

.. code:: python

    @update_hook.scan(period=5)
    async def update_hook(self, instance, async_lib):
        try:
            try:
                await self._update_server_state()
                if self._should_download():
                    await self._download_and_update()
            except (asyncio.TimeoutError,
                    aiohttp.client_exceptions.ClientConnectorError):
                self._consecutive_timeouts += 1
                self.log.warning('Timeout while updating server (%d in a row)',
                                 self._consecutive_timeouts)
                if self._consecutive_timeouts >= 6:
                    self.log.error('Too many consecutive timeouts!')
                    self._consecutive_timeouts = 0
                    raise
            else:
                self._consecutive_timeouts = 0
        except Exception:
            self.log.exception('Update failed!')
            for key, alarm in self.alarms.items():
                if key is not None:
                    await alarm.write(
                        status=AlarmStatus.COMM,
                        severity=AlarmSeverity.MAJOR_ALARM,
                    )


... get rid of PVGroup and pvproperty? I hate them!
---------------------------------------------------

Those are some strong words!

caproto developers think the ease of customization and terseness of the class
definition are positives - making it the easiest possible way to spin up an IOC
- but we respect that it's not a one-size-fits-all approach.

Let's rephrase the above as "How do I use the alternative methods of building
IOCs?"

The first option is to try working with ``PVSpec`` instances directly.
Take a look at the ``no_pvproperty`` example.

The second option is going down to the lowest level, working with
``ChannelData`` instances directly. Take a look at the advanced
``type_varieties`` example.


... represent both the setpoint and readback with one pvproperty?
-----------------------------------------------------------------

EPICS V3 records mostly do not have support for this, and caproto does not try
to improve upon the status quo.

A common convention -- areaDetector-style -- where there are 2 PVs, one
for the setpoint and one for the readback looks like the following:

.. code::

    SYSTEM:Value
    SYSTEM:Value_RBV


Where ``SYSTEM:Value`` is the setpoint, and ``SYSTEM:Value_RBV`` is the
"read-back value" that updates only after the server (or device) has
acknowledged it.

caproto provides a simple helper tool to replicate this pattern efficiently.
Using :func:`.server.get_pv_pair_wrapper`, one can dynamically generate a
``SubGroup`` that contains both the setpoint and readback PVs.  This wrapper is
re-usable and may be used to generate any number of such pairs.

.. code:: python

    pvproperty_with_rbv = get_pv_pair_wrapper(setpoint_suffix='',
                                              readback_suffix='_RBV')

    value = pvproperty_with_rbv(
        name="Value",
        value=0,
        doc="This is a new subgroup with Value and Value_RBV.",
    )

    @value.setpoint.putter
    async def value(obj, instance, value):
        # accept the value immediately
        await obj.readback.write(value)
        # NOTE: you can access the full Group instance through obj.parent


You can further customize the setpoint and readback keyword arguments
by using ``setpoint_kw`` and ``readback_kw``. For example, you can
set different record types by specifying ``setpoint_kw=dict(record="ao")``.

.. autosummary::
    :toctree: generated

    caproto.server.get_pv_pair_wrapper


.. note::

    The API of this changed in v0.7.0, making it easier to pass in class
    kwargs.


... do some really crazy things with caproto?
---------------------------------------------

Take a look at the pathological IOC examples.  Take extra care not to run them
in production!

... handle records and what are the limitations there?
------------------------------------------------------

For any given pvproperty, you can specify a caproto-supported record type by
way of the ``record=''`` keyword argument.

Please note that none of the classes listed here implement the full
functionality of the corresponding record, but make available over Channel
Access all of the fields one would normally expect from that record.

That said, some functionality is provided out-of-the-box, and you can further
customize this functionality in your own implementations.

See the :doc:`records` section for further details.

Further limitations/notes:

* In/out links do not work
* Many fields are not (yet) implemented

... add a client to monitor other PVs?
--------------------------------------

As of the time of writing, the threading client is more well-tested than the
asyncio client, and an example :mod:`.thread_client_monitor` was written
around it.

The above is admittedly rather complicated, but gives you a good amount of
control. It also demonstrates thread-to-async communication by way of
async library-independent queues.

A more user-friendly (but less tested) example is
:mod:`.ioc_examples.client_monitor_async`, which is currently limited to
asyncio-only:

.. literalinclude:: ../../caproto/ioc_examples/client_monitor_async.py

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.client_monitor_async.MirrorClientIOC


To work directly with the asyncio client context, you might consider basing
your IOC on the following example instead:

.. literalinclude:: ../../caproto/ioc_examples/mirror.py

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.mirror.Mirror


... get custom arguments into my IOC?
-------------------------------------

At the moment, the easiest way to get a custom argument into an IOC is by
specifying a macro with a default value to
:func:`caproto.server.ioc_arg_parser`.

These macros would be accessible in your ``PVGroup`` ``__init__`` as the
``macros`` keyword argument.

Alternatively, you can add arguments directly to the parser, as seen
in the ``chirp`` example:

.. code:: python

    parser, split_args = template_arg_parser(
        default_prefix="prefix:",
        desc="IOC description.",
    )

    parser.add_argument(
        "--flag",
        help="Set an arbitrary flag.",
        action='store_true',
    )

    args = parser.parse_args()
    ioc_options, run_options = split_args(args)
    ioc = MyPVGroup(flag=args.flag, **ioc_options)
    run(ioc.pvdb, **run_options)


If your needs are more advanced, feel free to reimplement the above functions.
So long as you can instantiate your PVGroup and pass its pvdb to ``run()``,
caproto will not complain.

... use macros? And what are macros?
------------------------------------

Macros allow you to specify portions of PV names at IOC initialization-time
instead of at PVGroup/pvproperty definition time.

Macros can be used in addition to - or in place of - the default PV prefix.

See the :class:`~.ioc_examples.macros.MacroifiedNames` for an example.

... make a bunch of caproto IOCs without all the boilerplate?
-------------------------------------------------------------

Take a look at the cookiecutters.  You can generate a full Python package IOC
template along with a startup script template pretty quickly with this tool.

... get a structured view of my PVGroup on the client side?
-----------------------------------------------------------

(This is a shameless plug for our other free tools...)

Consider trying `ophyd <https://blueskyproject.io/>`_. devices as the
client-side interface to your server-side caproto IOC.

If you buy into this ecosystem, you will be able to:

* Use your caproto-backed PVs in scans and other data acquisition routines
  by way of bluesky
* Auto-generate EPICS user interfaces with Typhos and PyDM
* Track and organize your devices by way of happi


Examples
========

Below, we will use caproto's threading client to interact with caproto IOC.

.. ipython:: python

    from caproto.threading.client import Context
    ctx = Context()  # a client Context used to explore the servers below

Of course, standard epics-base clients or other caproto clients may also be
used.

.. ipython:: python
    :suppress:

    import sys
    import subprocess
    import time
    processes = []
    def run_example(module_name, *args):
        p = subprocess.Popen([sys.executable, '-m', module_name] + list(args))
        processes.append(p)  # Clean this up at the end.
        time.sleep(1)  # Give it time to start up.

Simple IOC
----------

This IOC has two PVs that simply store a value.

.. literalinclude:: ../../caproto/ioc_examples/simple.py

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.simple.SimpleIOC


.. ipython:: python
    :suppress:

    run_example('caproto.ioc_examples.simple')

.. code-block:: bash

    $ python3 -m caproto.ioc_examples.simple --list-pvs
    [I 18:08:47.628 server:93] Server starting up...
    [I 18:08:47.630 server:109] Listening on 0.0.0.0:56840
    [I 18:08:47.630 server:121] Server startup complete.
    [I 18:08:47.630 server:123] PVs available:
        simple:A
        simple:B
        simple:C

Using the threading client context we created above, we can read these values
and write to them.

.. ipython:: python

    a, b, c = ctx.get_pvs('simple:A', 'simple:B', 'simple:C')
    a.read()
    b.read()
    c.read()
    b.write(5)
    b.read()
    c.write([4, 5, 6])
    c.read()

Write to a File When a PV is Written To
---------------------------------------

.. literalinclude:: ../../caproto/ioc_examples/custom_write.py

.. code-block:: bash

    $ python3 -m caproto.ioc_examples.custom_write --list-pvs
    [I 18:12:07.282 server:93] Server starting up...
    [I 18:12:07.284 server:109] Listening on 0.0.0.0:57539
    [I 18:12:07.284 server:121] Server startup complete.
    [I 18:12:07.284 server:123] PVs available:
        custom_write:A
        custom_write:B

.. ipython:: python
    :suppress:

    run_example('caproto.ioc_examples.custom_write')

On the machine where the server resides, we will see files update whenever any
client writes.

.. ipython:: python

    a, b = ctx.get_pvs('custom_write:A', 'custom_write:B')
    a.write(5)
    print(open('/tmp/A').read())
    a.write(10)
    print(open('/tmp/A').read())

It is easy to imagine extending this example to write a socket or a serial
device rather than a file.

Random Walk
-----------

This example contains a PV ``random_walk:x`` that takes random steps at an
update rate controlled by a second PV, ``random_walk:dt``.

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.random_walk.RandomWalkIOC

.. literalinclude:: ../../caproto/ioc_examples/random_walk.py

.. note::

   **What is async_lib.library.sleep?**

   As caproto supports three different async libraries, we have an "async
   layer" class that gives compatible async versions of commonly-used
   synchronization primitives. The attribute ``async_lib.library`` would be either
   the Python module ``asyncio`` (default), ``trio``, or ``curio``, depending on how
   the server is run.  It happens that all three of these modules have a ``sleep``
   function at the top level, so ``async_lib.library.sleep`` accesses the
   appropriate sleep function for each library.

   **Why not use time.sleep?**

   The gist is that ``asyncio.sleep`` doesn't hold up your entire thread /
   event loop, but gives back control to the event loop to run other tasks while
   sleeping. The function ``time.sleep``, on the other hand, would cause
   noticeable delays and problems.

   This is a fundamental consideration in concurrent programming generally,
   not specific to caproto. See for example
   `this StackOverflow post <https://stackoverflow.com/questions/46207991/what-does-yield-from-asyncio-sleepdelay-do>`_
   for more information.


I/O Interrupt
-------------

This example listens for key presses.

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.io_interrupt.start_io_interrupt_monitor
    caproto.ioc_examples.io_interrupt.IOInterruptIOC


.. literalinclude:: ../../caproto/ioc_examples/io_interrupt.py

.. code-block:: bash

    $ python -m caproto.ioc_examples.io_interrupt --list-pvs
    [I 10:18:57.643 server:132] Server starting up...
    [I 10:18:57.644 server:145] Listening on 0.0.0.0:54583
    [I 10:18:57.646 server:218] Server startup complete.
    [I 10:18:57.646 server:220] PVs available:
        io:keypress
    * keypress method called at server startup
    Started monitoring the keyboard outside of the async library

Typing causes updates to be sent to any client subscribed to ``io:keypress``.
If we monitoring using the commandline-client like so:

.. code-block:: bash

    $ caproto-monitor io:keypress

and go back to the server and type some keys:

.. code-block:: bash

    New keypress: 'a'
    Saw new value on async side: 'a'
    New keypress: 'b'
    Saw new value on async side: 'b'
    New keypress: 'c'
    Saw new value on async side: 'c'
    New keypress: 'd'
    Saw new value on async side: 'd'

the client will receive the updates:

.. code-block:: bash

    io:keypress                               2018-06-14 10:19:04 [b'd']
    io:keypress                               2018-06-14 10:20:26 [b'a']
    io:keypress                               2018-06-14 10:20:26 [b's']
    io:keypress                               2018-06-14 10:20:26 [b'd']

Macros for PV names
-------------------

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.macros.MacroifiedNames


.. literalinclude:: ../../caproto/ioc_examples/macros.py

The help string for this IOC contains two extra entries at the bottom:

.. code-block:: bash

    $ python3 -m caproto.ioc_examples.macros -h
    usage: macros.py [-h] [--prefix PREFIX] [-q | -v] [--list-pvs]
                    [--async-lib {asyncio,curio,trio}]
                    [--interfaces INTERFACES [INTERFACES ...]]
                    [--beamline BEAMLINE] [--suffix SUFFIX]

    Run an IOC with PVs that have macro-ified names.

    optional arguments:

    <...snipped...>

    --beamline BEAMLINE   Macro substitution, optional
    --suffix SUFFIX       Macro substitution, optional

.. code-block:: bash

    $ python3 -m caproto.ioc_examples.macros --beamline XF31ID --suffix detector --list-pvs
    [I 18:44:39.528 server:93] Server starting up...
    [I 18:44:39.530 server:109] Listening on 0.0.0.0:56365
    [I 18:44:39.531 server:121] Server startup complete.
    [I 18:44:39.531 server:123] PVs available:
        macros:XF31ID:detector.VAL
        macros:XF31ID:detector.RBV

.. ipython:: python
    :suppress:

    run_example('caproto.ioc_examples.macros', '--beamline', 'XF31ID', '--suffix', 'detector')

.. ipython:: python

    pv, = ctx.get_pvs('macros:XF31ID:detector.VAL')
    pv.read()

Observe that the command line arguments fill in the PV names.

Subgroups
---------

The PVGroup is designed to be nested, which provides a nice path toward future
capability: IOCs that are natively "V7", encoding semantic structure for PV
Access clients and decomposing into flat PVs for Channel Access clients.

.. literalinclude:: ../../caproto/ioc_examples/subgroups.py

.. code-block:: bash

    $ python -m caproto.ioc_examples.subgroups --list-pvs
    random using the descriptor getter is: <caproto.server.server.PvpropertyInteger object at 0x10928ffd0>
    subgroup4 is: <__main__.MyPVGroup.group4.subgroup4 object at 0x10928fe10>
    subgroup4.random is: <caproto.server.server.PvpropertyInteger object at 0x10928fef0>
    [I 11:08:35.074 server:132] Server starting up...
    [I 11:08:35.074 server:145] Listening on 0.0.0.0:63336
    [I 11:08:35.076 server:218] Server startup complete.
    [I 11:08:35.077 server:220] PVs available:
        subgroups:random
        subgroups:RECORD_LIKE1.RTYP
        subgroups:RECORD_LIKE1.VAL
        subgroups:RECORD_LIKE1.DESC
        subgroups:recordlike2.RTYP
        subgroups:recordlike2.VAL
        subgroups:recordlike2.DESC
        subgroups:group1:random
        subgroups:group2-random
        subgroups:group3_prefix:random
        subgroups:group4:subgroup4:random

.. _records_example:

Records
-------

See :doc:`records`.

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.records.RecordMockingIOC


.. literalinclude:: ../../caproto/ioc_examples/records.py

.. code-block:: bash

    python -m caproto.ioc_examples.records --list-pvs
    PVs: ['mock:A', 'mock:B']
    Fields of B: ['ACKS', 'ACKT', 'ASG', 'DESC', 'DISA', 'DISP', 'DISS', 'DISV', 'DTYP', 'EVNT', 'FLNK', 'LCNT', 'NAME', 'NSEV', 'NSTA', 'PACT', 'PHAS', 'PINI', 'PRIO', 'PROC', 'PUTF', 'RPRO', 'SCAN', 'SDIS', 'SEVR', 'TPRO', 'TSE', 'TSEL', 'UDF', 'RTYP', 'STAT', 'RVAL', 'INIT', 'MLST', 'LALM', 'ALST', 'LBRK', 'ORAW', 'ROFF', 'SIMM', 'SVAL', 'HYST', 'HIGH', 'HSV', 'HIHI', 'HHSV', 'LOLO', 'LLSV', 'LOW', 'LSV', 'AOFF', 'ASLO', 'EGUF', 'EGUL', 'LINR', 'EOFF', 'ESLO', 'SMOO', 'ADEL', 'PREC', 'EGU', 'HOPR', 'LOPR', 'MDEL', 'INP', 'SIOL', 'SIML', 'SIMS']
    [I 11:07:48.635 server:132] Server starting up...
    [I 11:07:48.636 server:145] Listening on 0.0.0.0:49637
    [I 11:07:48.638 server:218] Server startup complete.
    [I 11:07:48.638 server:220] PVs available:
        mock:A
        mock:B


Mini-Beamline
-------------

Simulate your own mini beamline with this IOC.

.. autosummary::
    :toctree: generated

    caproto.ioc_examples.mini_beamline
    caproto.ioc_examples.mini_beamline.MiniBeamline
    caproto.ioc_examples.mini_beamline.PinHole
    caproto.ioc_examples.mini_beamline.Edge
    caproto.ioc_examples.mini_beamline.Slit
    caproto.ioc_examples.mini_beamline.MovingDot


More...
-------

Take a look around
`the ioc_examples subpackage <https://github.com/caproto/caproto/tree/master/caproto/ioc_examples>`_ for more examples not covered here.


.. autosummary::
    :toctree: generated

    caproto.ioc_examples.enums.EnumIOC
    caproto.ioc_examples.autosave.AutosavedSimpleIOC
    caproto.ioc_examples.decay.Decay
    caproto.ioc_examples.scan_rate.ScanRateIOC


.. ipython:: python
    :suppress:

    # Clean up IOC processes.
    for p in processes:
        p.terminate()
    for p in processes:
        p.wait()

Helpers
=======

caproto offers several "helper" subgroups (:class:`~server.SubGroup`)
that are of general use, and could be considered part of the "caproto server
standard library", so to speak.

Autosave
--------

.. currentmodule:: caproto.server.autosave

.. autosummary::
    :toctree: generated

    AutosaveHelper
    RotatingFileManager


Status / Statistics
-------------------

.. currentmodule:: caproto.server.stats

.. autosummary::
    :toctree: generated

    StatusHelper
    BasicStatusHelper
    PeriodicStatusHelper
    MemoryTracingHelper
