Maintainers guide
=================


Release process
---------------

All OSBS projects use the same release model. Examples are based on koji-container package.

Build a package
~~~~~~~~~~~~~~~

To build the current release, use the following command in the repo directory::

  tito build --rpm


Create new release
~~~~~~~~~~~~~~~~~~

Please note that regular releases are done from master branch and hotfix/patch releases are done from release branch.

Create upstream release
***********************

In this upstream repository:

1. Bump release and commit changelog. It needs to be without release part, only dot separated one (for example 0.5.7)::

    tito tag --use-version 0.5.7

2. See last line of tito output which give you a hint how to push commit and tag to remote (for example 0.5.7-1)::

    git push origin
    git push origin koji-containerbuild-0.5.7-1


3. Create srpm::

    tito build --srpm

Update downstream release
*************************

Following steps are for updating packages in Fedora:

1. Clone or pull latest changes in downstream repository::

    fedpkg co koji-containerbuild

2. Switch to branch which you want to update (e.g. start with master)::

    fedpkg switch-branch master

3. Import srpm created in previos section (for example koji-containerbuild version 0.5.7-1, updating Fedora 24)::

    fedpkg import /tmp/tito/koji-containerbuild-0.5.7-1.f24.src.rpm

  Review changes - there should be single new tarball, entries in changelog looks fine (e.g. not empty).

4. Try to build package as scratch-build::

    fedpkg scratch-build --srpm

5. If build succeeded commit changes to remote and build regular build::

    fedpkg push
    fedpkg build

Update other branches either by merging (preferred) or importing srpm.
