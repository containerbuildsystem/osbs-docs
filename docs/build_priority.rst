Priority of Container Image Builds
==================================

For a build system it's desirable to prioritize different kinds of builds in
order to better utilize resources. Unfortunately, OpenShift's scheduling
algorithm does not support setting a priority value for a given build. To
achieve some sort of build prioritization, we can leverage node selectors to
allocate different resources to different build types.

Consider the following types of container builds builds:

- *scratch build*
- *explicit build*
- *auto rebuild*

As the name implies, *scratch builds* are meant to be used as a one-off
unofficial container build. No guarantees are made for storing the created
container images long term. It’s also not meant to be shipped to customers.
These are clearly low priority builds.

*Explicit builds* are those triggered by a user, either directly via fedpkg/koji
CLI, or indirectly via pungi (as in the case of base images). These are official
builds that will go through the normal life cycle of being tested and,
eventually, shipped.

*Auto rebuilds* are created by OpenShift when a change in the parent image is
detected. It’s likely that layered images should be rebuilt in order to pick up
changes in latest parent image.

For any *explicit build* or *auto rebuild*, they may or may not be high
priority. In some cases, a build is high priority due to a security fix, for
instance. In other cases, it could be due to an in-progress feature. For this
reason, it cannot be said that all *explicit builds* are higher priority than
*auto rebuilds*, or vice-versa.

However, *auto rebuilds* will be enabled by a new feature that has the potential
of completely consuming OSBS’s infrastructure. There must be some mechanism to
throttle the amount of *auto rebuilds*. For this reason, OSBS uses a different
node selector for each different build type:

- *scratch build*: builds_scratch=true
- *explicit build*: builds_explicit=true
- *auto rebuild*: builds_auto=true

By controlling each type of builds individually, OSBS will have the necessary
control for adjusting its infrastructure.

For example, consider an OpenShift cluster with 5 compute nodes:

======  =================== ==================== ================
Node    builds_scratch=true builds_explicit=true builds_auto=true
======  =================== ==================== ================
Node 1  ✔                   ✔                    ✔
Node 2  ✗                   ✔                    ✗
Node 3  ✗                   ✗                    ✔
Node 4  ✗                   ✔                    ✔
Node 5  ✗                   ✔                    ✔
======  =================== ==================== ================

In this case, *scratch builds* can be scheduled only on **Node 1**; *explicit
builds* on any node except **Node 3**; and auto builds on any node except **Node
2**.

Worker Builds Node Selectors
----------------------------

The build type node selectors are only applied to worker builds. This gives a
more granular control over available resources. Since worker builds are the ones
that actually perform the container image building steps, it requires more
resources than orchestrator builds. For this reason, a deployment is more likely
to have more nodes available for worker builds than orchestrator builds. This is
important because the amount of nodes available defines the granularity of how
builds are spread across the cluster.

For instance, consider a large deployment in which only 2 orchestrator nodes are
needed.  If build type node selectors are applied to orchestrator builds, builds
can only be throttled by a factor of 2. In contrast, this same deployment may
use 20 worker builds, allowing builds to be throttled by a factor of 20.

Orchestrator Builds Allocation
------------------------------

Usually in a deployment, the amount of allowed orchestrator builds matches the
amount of allowed worker builds for any given platform. Additional orchestrator
builds should be allowed to fully leverage the build type node selectors on
worker builds since some orchestrator builds will wait longer than usual for
their worker builds to be scheduled. This provides a buffer that allows
OpenShift to properly schedule worker builds according to their build type via
node selectors. Because OpenShift scheduling is used, worker builds of same type
will run in the order they were submitted.


Koji Builder Capacity
--------------------------

The task load of the Koji builders used by OSBS will not reflect the actual load
on the OpenShift cluster used by OSBS. The disparity is due to auto rebuilds not
having a corresponding Koji task. This creates a scenario where a buildContainer
Koji task is started, but the OpenShift build remains in pending state. The Koji
builder capacity should be set based on how many nodes allow **scratch builds**
and/or **excplicit builds**. In the example above, there are 4 nodes that allow
such builds.

Providing the osbs-client logs in the Koji task should give users a better
understanding of any delays due to scheduling.
