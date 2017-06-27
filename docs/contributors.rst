Contributing to OSBS
====================

Setting up a (local) development environment
--------------------------------------------

Contribution guidelines
-----------------------

Submitting changes
~~~~~~~~~~~~~~~~~~

Changes are accepted through pull requests. If you want to implement
major new functionality, we'd like to ask you to open a bug with
proposal to discuss it with us. We're nice folks and we don't bite --
we just want to see what you're up to and a chance to give some
suggestions to save your time as well as ours.

Please create your feature branch from the master branch. Make sure to
add unit tests under the tests/ subdirectory (we use py.test and
flexmock for this). When you push new commits, tests will be triggered
to be run in Travis CI and results will be shown in your pull
request. You can also run them locally from the top directory (py.test
tests).

Follow the PEP8 coding style. This project allows 99 characters per
line.

Please make sure each commit is for a complete logical change, and has
a useful commit message. When making changes in response to
suggestions, it is fine to make new commits for them but please make
sure to squash them into the relevant "logical change" commit before
the pull request is merged.

Before a pull request is approved it must meet these criteria:

- unit tests pass

- code coverage from testing does not decrease and new code is covered

- new features must have user documentation

Once it is approved by two developer team members someone from the
team will merge it. To avoid creating merge commits the pull request
will be rebased during the merge.

Licensing
~~~~~~~~~

This project is licensed using the BSD-3-Clause license. When
submitting pull requests please make sure your commit messages include
a signed-off-by line to certify the below text::

  Developer's Certificate of Origin 1.1

  By making a contribution to this project, I certify that:

  (a) The contribution was created in whole or in part by me and I
      have the right to submit it under the open source license
      indicated in the file; or

  (b) The contribution is based upon previous work that, to the best
      of my knowledge, is covered under an appropriate open source
      license and I have the right under that license to submit that
      work with modifications, whether created in whole or in part
      by me, under the same open source license (unless I am
      permitted to submit under a different license), as indicated
      in the file; or

  (c) The contribution was provided directly to me by some other
      person who certified (a), (b) or (c) and I have not modified
      it.

  (d) I understand and agree that this project and the contribution
      are public and that a record of the contribution (including all
      personal information I submit with it, including my sign-off) is
      maintained indefinitely and may be redistributed consistent with
      this project or the open source license(s) involved.

You can do this by using git commit --signoff when committing changes,
making sure that your real name is included.

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
