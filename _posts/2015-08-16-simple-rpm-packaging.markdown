---
title: "RPM Packaging"
date:  2015-08-16 17:26:00
description: Keeping packaging simple while following best practices.
keywords: simple, rpm, packaging, linux, lookaside cache, spkg, vcs-friendly, fedpkg, koji, obs, package management
---

Sooner or later when working with a fleet of linux servers, it becomes necessary to package software. A few reasons to build native packages:

* They play nice with configuration management.
* File integrity checks come for free with package managers.
* Any sysadmin can find out where a particular file came from.
* Scriptlets.

The bulk of my packaging experience has been on RPM-based systems where there are a few standout options/frameworks for package building, such as:

* [Fedpkg](https://fedorahosted.org/fedpkg/) with [Koji](https://fedorahosted.org/koji/wiki)
* [openSUSE Build Service](https://build.opensuse.org/)
* [CentOS Sources](http://wiki.centos.org/Sources)
* Project-specific or DIY one-off build scripts.

The first two of these options use the notion of a lookaside cache. This basically means putting binary or other non vcs-friendly files onto another server and simply referring to them. As far as I can tell, OBS bundles together source binaries rather than referring to them in a separate cache. However, given that all of the files for OBS are kept in a cloud system, there are some similarities to be drawn.

Not everybody has the infrastructure for a dedicated Koji or lookaside cache. It's also a lot of additional work for a sysadmin who wants to build some RPMs and has a lot of other things on their plate.

The other end of the spectrum is the one-off <code>build.sh</code> style scripts. These are quick to put in place, but slow down development when the need comes for adding additional sources or other complexity (like gpg signing).

Does this look familiar?

```bash
#!/bin/bash
cp source1 source2 ~/rpmbuild/SOURCES/
rpmbuild -ba myproject.spec
```

## A compromise

I decided to write a script that would offer a time/complexity trade-off between the dedicated build system and the one-off script. Some design goals:

* Not tied to a public system (cloud or otherwise). This is two-fold:
    * Corporate-friendly - all the backends can be kept internal.
    * Ability to fork existing packages without having to worry about merging back/hosting/etc. Unless you want to, and more power to you at that point.
* Easy to install and quick to see results.
* Keeping binaries out of the version controlled build environment, but ensuring their integrity when used.

## <code>spkg</code> - Simple packaging for RPMs

<code>spkg</code> works around a build environment that is at its core, a directory containing

* An [RPM spec](https://fedoraproject.org/wiki/How_to_create_an_RPM_package#Examples) detailing the steps required to build one or more RPMs.
* A <code>sources</code> directory containing all non-remote sources referred to in the spec file (required if there are any local sources).
* A <code>checksums</code> file listing the sha256 sums of remote sources referred to in the spec file (optional but encouraged).

The build environment should be kept under version control.

<code>spkg</code> can create this hierarchy automatically from an existing SRPM:

```
$ spkg init ~/srpm-test/nginx-1.8.0-1.el7.ngx.src.rpm
...
New build directory under nginx-1.8.0-1.el7.ngx.src
$ tree nginx-1.8.0-1.el7.ngx.src/
nginx-1.8.0-1.el7.ngx.src/
├── checksums
├── nginx.spec
└── sources
    ├── logrotate
    ├── nginx.conf
    ├── nginx.init
    ├── nginx.service
    ├── nginx.suse.init
    ├── nginx.suse.logrotate
    ├── nginx.sysconf
    ├── nginx.upgrade.sh
    ├── nginx.vh.default.conf
    └── nginx.vh.example_ssl.conf
```

Or you can put a spec file in a directory and <code>spkg</code> will figure out the details, prompting you when it's missing something.

When you're ready to build, execute the following command inside the build environment.

```bash
spkg build
```

For more details, take a look at the [project on Github](https://github.com/mdsummers/spkg).

Have I missed something? Send a pull request or create an issue.

### See also
* [Fedpkg presentation (pdf)](https://fedoraproject.org/w/uploads/1/1c/Fedpkg-presentation.pdf)
* [Package maintenance guide - FedoraProject](https://fedoraproject.org/wiki/Package_maintenance_guide)
* [CentOS git repos](https://git.centos.org/)
