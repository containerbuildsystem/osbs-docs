Contributing to OSBS
====================

Setting up a (local) development environment
--------------------------------------------

Contribution guidelines
-----------------------

Code policies
~~~~~~~~~~~~~

Retry on error
..............

Requests to a remote service should be retried if the failure is not
definitely permanent.

In most cases there should be a delay before retrying. The only reason
to retry immediately is when the request is likely to succeed when
immediately retried (i.e. the cause is known). An example would be
retrying a PUT operation which was rejected due to conflicts.

There should be a maximum number of attempts before giving up and
reporting an exception.

The delay and the maximum number of attempts should both be
configurable if feasible. If not explicitly configured and an HTTP
request to be retried included a Retry-After header in the response,
the Retry-After header should be used to set the delay time.

When alternate remote services are available, retries should only be
attempted if no alternates succeed. For example, when selecting a
worker cluster a failure should immediately result in the next worker
cluster being tried. If there is no such cluster, or all configured
worker clusters fail, retries should be attempted, including a delay
if appropriate (depending on the failure).

There should only be a single "level" of retry logic, unless there is
a good reason. For example, if dockpulp implements retry logic for one
of its operations, atomic-reactor should not retry that operation
(unless it makes sense to). In other words: be aware of which calls
implicitly retry.
