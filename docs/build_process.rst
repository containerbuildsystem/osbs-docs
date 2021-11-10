.. _`build process`:

Understanding the Build Process
===============================

Logging
-------

Logs from builds is made available via the orchestrator build,
and clients (including koji-containerbuild) are able to separate
individual build logs out from that log stream using an
osbs-client API method.

Multiplexing
~~~~~~~~~~~~

In order to allow the client to de-multiplex logs containing a mixture
of logs from an orchestrator build and from its worker builds, a
special logging field, platform, is used. Within atomic-reactor all
logging goes through a LoggerAdapter which adds this ``platform``
keyword to the ``extra`` dict passed into logging calls, resulting in
log output like this:

::

    2017-06-23 17:18:41,791 platform:- - atomic_reactor.foo - DEBUG - this is from the pipeline task
    2017-06-23 17:18:41,791 platform:x86_64 - atomic_reactor.foo - INFO - 2017-06-23 17:18:41,400 platform:- atomic_reactor.foo - DEBUG - this is from a build
    2017-06-23 17:18:41,791 platform:x86_64 - atomic_reactor.foo - INFO - continuation line

Demultiplexing is possible using a the osbs-client API method,
``get_orchestrator_build_logs``, a generator function that returns
objects with these attributes:

platform
  str, platform name if worker build, else None

line
  str, log line (Unicode)

The "Example" section below demonstrates how to use the
``get_orchestrator_build_logs()`` method in Python to parse the above log
lines.

Encoding issues
~~~~~~~~~~~~~~~

When retrieving logs from containers, the text encoding used is only
known to the container. It may be based on environment variables
within that container; it may be hard-coded; it may be influenced by
some other factor. For this reason, container logs are treated as byte
streams.

When retrieving logs from a build, OpenShift cannot say which encoding
was used. However, atomic-reactor can define its own output encoding
to be UTF-8. By doing this, all its log output will be in a known
encoding, allowing osbs-client to decode it. To do this it should call
``locale.setlocale(locale.LC_ALL, "")`` and the Dockerfile used to
create the builder image must set an appropriate environment
variable::

  ENV LC_ALL=en_US.UTF-8


Example
~~~~~~~

Here is an example Python session demonstrating this interface::

  >>> server = OSBS(...)
  >>> logs = server.get_build_logs(...)
  >>> [(item.platform, item.line) for item in logs]
  [(None, '2017-06-23 17:18:41,791 platform:- - atomic_reactor.foo - DEBUG - this is from the pipeline task'),
   ('x86_64', '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'),
   ('x86_64', 'continuation line')]

Note:

- the lines are (Unicode) string objects, not bytes objects

- the orchestrator build's logging fields have been removed from the
  worker build log line

- the "outer" orchestrator log fields have been removed from the
  worker build log line, and the ``platform:-`` field has also been
  removed from the worker build's log line

- where the worker build log line had no timestamp (perhaps the log
  line had an embedded newline, or was logged outside the adapter
  using a different format), the line was left alone


