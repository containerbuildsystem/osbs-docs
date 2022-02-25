.. _`build process`:

Understanding the Build Process
===============================

Logging
-------

Logs from builds are made available via osbs-client API,
and clients (including koji-containerbuild) are able to separate
individual task logs out from that log stream using an
osbs-client API method.

Getting logs during build
~~~~~~~~~~~~~~~~~~~~~~~~~

Logs can be streamed from the build via osbs-client API method
``get_build_logs`` and setting ``follow`` and ``wait`` parameters
to ``True``.

These logs are returned in tuple form as ``(task_run_name, log_line)``. Where
``task_run_name`` is the name of the task that generated the log line. The name
of the task will also contain the platform that task is building for if it is a
platform specific task. This can be used to separate the logs by platform.

Getting logs after the build
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``get_build_logs`` of osbs-client API will return the logs as a dictionary. The
top level keys will be the task name that can also be used to identify platform
specific logs and separate them into platform specific log files.

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

Here is an example Python session demonstrating this interface for streaming::

  >>> server = OSBS(...)
  >>> logs = server.get_build_logs(follow=True, wait=True)
  >>> [(task_run_name, log_line) for task_run_name, log_line in logs]
  [('binary-container-prebuild', '2017-06-23 17:18:41,791 platform:- - atomic_reactor.foo - DEBUG - this is from the pipeline task'),
   ('binary-container-build-x86-64', '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'),
   ('binary-container-build-aarch64', '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'),
   ('binary-container-build-s390x', '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'),
   ('binary-container-build-ppc64le', '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'),
   ('binary-container-postbuild', 'continuation line')]

Note:

- the lines are (Unicode) string objects, not bytes objects

- where the build log line had no timestamp (perhaps the log
  line had an embedded newline, or was logged outside the adapter
  using a different format), the line was left alone

Here is an example Python session demonstrating this interface non-streaming::

  >>> server = OSBS(...)
  >>> logs = server.get_build_logs()
  >>> logs
  {'binary-container-prebuild': {'containerA': '2017-06-23 17:18:41,791 platform:- - atomic_reactor.foo - DEBUG - this is from the pipeline task'},
   'binary-container-build-x86-64': {'containerB': '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'},
   'binary-container-build-aarch64': {'containerC': '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'},
   'binary-container-build-s390x': {'containerD': '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'},
   'binary-container-build-ppc64le': {'containerE': '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a build'},
   'binary-container-postbuild': {'containerF': 'continuation line'}}


